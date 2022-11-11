---
typora-root-url: pic
---

# Mutex的实现

- 虽然go鼓励用通信的方式来共享内存，但是用共享内存的方式来通信依然是不可或缺的。
- 比如某一个变量或者某一段代码，可能会被多个协程同时执行，但是我又只希望同一时刻只有一个协程能操作，就需要用到互斥锁。这个变量或者这段代码称为临界区。每一个协程在执行临界区代码前，都需要使用互斥锁的Lock方法尝试抢执行权，如果抢不到会阻塞住，直到抢到了才能往后执行。当执行完后需要使用互斥锁的UnLock方法将执行权交出去，其他协程才有可能抢到执行权。
- 本文的逻辑是基于`鸟窝`大佬对锁源码分析的文章，一步步详细分析go标准库的Mutex的原理。从而做到让自己尽可能的避免在使用过程中踩到莫名奇妙的坑。



在开始实现互斥锁时，先要了解CAS

## CAS

- CAS(compare and swap)意思是比较并交换。

- 以go标准库中的atomic.CompareAndSwapInt32方法为例

  ```go
  // CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
  func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
  ```

  - 它接收三个入参，一个指针addr，一个旧值old，一个新值new
  - 这个方法做的事情分为三步:
    - 首先取addr指向的值
    - 然后与old比较，如果不相等，返回false，操作失败。
    - 最后将addr指向的值改成new
  - 关键在于，机器层面保证了这三步的原子性，既执行过程中不会被其他协程所打断。而普通代码可能在执行过程中突然被打断，cpu转而去执行其他协程了。
  - 除了CAS，go的atomic包还提供了其他原子方法，原子方法可以解决一些并发场景下的数据竞争问题。

## 开始实现互斥锁

### v0版本

```go
package v0

import "sync/atomic"

type Mutex struct {
	key int32
}

func (m *Mutex) Lock(){
	for {
		if atomic.CompareAndSwapInt32(&m.key,0,1) {
			return
		}
	}
}

func (m *Mutex) Unlock() {
    atomic.CompareAndSwapInt32(&m.key,1,0) 
}
```

#### Mutex

- 包含key字段，来记录锁的状态

#### Lock

- for循环不断的尝试CAS操作
  - 假设有两个协程g1,g2在竞争同一把锁
  - g1先执行，拿到了锁，在操作临界资源时，g2来拿锁了
  - 这时g2的Lock方法会尝试执行CAS，但是在CAS的第二步中，会失败，因为g1已经把key改成1了。g2会继续尝试CAS，所以g2阻塞在此。
  - g1对临界资源的操作完毕，调用了Unlock方法，通过CAS将key值改回了0。
  - 之后g2的Lock方法才能成功。

#### 问题

- 虽然实现了需求，但是毫无性能可言。没抢到锁的g2什么正事也干不了，但是一直吃着cpu的资源。

  

## mutexV1

- 这版本的目的，就是设法让拿不到锁的goroutine休眠，别再浪费资源了。

### mutex结构

```go
type Mutex struct {
	key int32
    sema int32
}
```

- key
  - 它的值不在是0和1,
  - 0表示锁未被持有，1表示锁已经被持有，n表示锁被持有并且有n-1个锁的等待者
- sema
  - 表示信号量
  - 用于休眠与唤醒，示例代码中，使用以下伪代码表示功能，但在go runtime中都有对应方法。
    - semacquire(sema)
      - 将当前goroutine休眠
    - semrelease(sema)
      - 唤醒从使用sema信号量休眠的协程队列中取出第一个协程

Lock方法和Unlock方法不在是对key做0和1之间的替换了，而是采用±1操作，这样不仅能记录锁的持有状态，还能记录排队的goroutine数

将±1操作抽出了xadd函数

### xadd

```go
func xadd(val *int32, delta int32) (new int32) {
	for {
		v := *val
		// val 表示地址
		// v 表示旧值
		// v+delta 表示新值
		if atomic.CompareAndSwapInt32(val,v,v+delta) {
			return v + delta
		}
	}
	panic("unreached")
}
```

- for循环中通过CAS改变val的值，val是key，delta是±1
- 这里也有for循环，for循环是因为可能存在有多个goroutine同时在竞争CAS操作

  - 从`v := *val`到CAS之间可不是原子操作，有可能刚刚拿到的v=1，等到执行到CAS这一行时，被其他执行这段逻辑的协程改成2了。CAS就成功不了，所以需要for循环。
- 这个for循环和v0版本的for循环不同

  - v0的目的是0和1之间做交换，抢锁时从0到1，而抢锁失败的goroutine的需要在for循环中等待已经有锁的goroutine释放锁。


### Lock

```go
func (m *Mutex) Lock() {
	// 通过CAS将key的值加1
	if xadd(&m.key,1) == 1 {
		return
	}
	// 将该goroutine阻塞等待
    semacquire(&m.sema)
	return
}
```

- xadd执行完后，key值如果等于1，那证明当前goroutine已经拿到了锁，可以直接返回执行后续临界区代码了。
- 如果key值大于1，那证明已经拿到了锁的待使用资格，不过还有其他goroutine也拿到了待使用资格，得去排队。
- 这里排队的方式是将自己休眠，而不是v0那样疯狂循环消耗CPU资源。

### UnLock

```go
func (m *Mutex) Unlock() {
	// 通过CAS将key的值减1
	if xadd(&m.key,-1) == 0 {
		return
	}
	// 唤醒阻塞中的goroutine
    semrelease(&m.sema)
	return
}
```

- Unlock方法就是需要将key减去1，如果操作完后key==0，不用管了直接返回。
- 但是执行临界区代码的这段时间内，可能有很多goroutine对key做了+1的操作，然后在排队。
- key大于0时，证明有休眠排队的goroutine，它们需要被唤醒，所以Unlock不能偷偷的返回。

### 问题

-  v1版本中，抢锁的众多goroutine中，不是第一次抢到都会经过休眠。
- 休眠再唤醒，涉及到协程间切换，这也是有损性能的。
- 能不能在Unlock解锁的那一瞬间，如果发现有正在请求锁的活跃goroutine，直接把锁给它，这就少了一次休眠在唤醒的过程，有助于提升性能。



## mutexV2

### mutex结构

```go
   type Mutex struct {
        state int32
        sema  uint32
    }


    const (
        mutexLocked = 1 << iota // mutex is locked // 1
        mutexWoken //2
        mutexWaiterShift = iota //2
    )
```

- state
  - 由key变成了state，state的类型是int32，由32个bit组成
  - 32bit的state是一个复合型字段，可分为三个部分 mutexWaiters | mutexWoken | mutexLocked
    - mutexWaiters表示锁的等待者数量
    - mutexWoken表示锁是否有唤醒的goroutine
    - mutexLocked表示锁是否被持有
  - 它的运算方式见文末”Mutex中的位运算逻辑“部分
- sema
  - 依然做为goroutine休眠与唤起的信号量

### Lock

```go
func (m *Mutex) Lock() {
        // Fast path: 幸运case，能够直接获取到锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }

        awoke := false
        for {
            old := m.state
            new := old | mutexLocked // 新状态加锁
            if old&mutexLocked != 0 {
                new = old + 1<<mutexWaiterShift //等待者数量加一
            }
            if awoke {
                // goroutine是被唤醒的，
                // 新状态清除唤醒标志
                new &^= mutexWoken
            }
            // 这个CAS的竞争者是唤醒的goroutine和新的goroutine
            if atomic.CompareAndSwapInt32(&m.state, old, new) {//设置新状态
                if old&mutexLocked == 0 { // 锁原状态未加锁
                    break
                }
                runtime.Semacquire(&m.sema) // 请求信号量
                awoke = true
            }
        }
    }
```

- 抢锁时最好的情况就是原state等于0，只需要通过CAS将state替换成mutexLocked就可以了，成功直接返回。要注意这里没有for循环了，没抢到直接进入后面的步骤。
- 其他情况就需要去竞争了

  - 首先是构造用于一会用来替换旧state的新值
    - `new := old | mutexLocked `新值首先需要加上锁持有状态
    - `if old&mutexLocked != 0`如果原本就是加过锁状态的，那说明当前goroutine要排队，所以需要给等待者数量加1
      - `new = old + 1<<mutexWaiterShift`
    - awoke的判断先不要看，看完Unlock逻辑再回过头思考
      - 只要进入了这块代码，说明自己是被唤醒的goroutine，已经醒了并且开始，就把这个状态清除掉。
  - 然后通过CAS，用新值替换旧值
  
    - 成功
  
      - `if old&mutexLocked == 0`。此时发现加锁状态没了，直接break。这是区别v1的主要逻辑。
        - v2版本Lock时，锁的状态只有三种可能：
          - 锁未被持有，且无等待者
          - 锁未被持有，但有等待者
          - 锁已被持有
        - 对于第一种情况，当前Lock的goroutine直接拿到锁
        - 对于第二种情况，在v1中虽然没有等待者状态的计数，但是它们是客观存在的，且都是休眠状态的。如果按照v1的思路，当前goroutine应该进入休眠，并且唤醒一个等待者。但是v2版本的优化就是要在抢锁过程中，让活跃的goroutine的优先级高于休眠的goroutine。所以这种情况，不好意思，我要`插队`了，当前goroutine直接拿到锁。
      - 第三钟情况在if语句之外。如果此时锁的状态是被持有，当前goroutine不可避免还是得去休眠，同时将awoke改为true，表示当前goroutine休眠后有朝一日再次被唤起后，已经是一个被唤醒过的goroutine了。
    - 失败
    
      - 没啥好说的，继续取值，继续构造旧值，继续CAS

### Unlock

```go
func (m *Mutex) Unlock() {
        // Fast path: drop lock bit.
        new := atomic.AddInt32(&m.state, -mutexLocked) //去掉锁标志
        if (new+mutexLocked)&mutexLocked == 0 { //本来就没有加锁
            panic("sync: unlock of unlocked mutex")
        }
    
        old := new
        for {
            if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0 { // 没有等待者，或者有唤醒的waiter，或者锁原来已加锁
                return
            }
            new = (old - 1<<mutexWaiterShift) | mutexWoken // 新状态，准备唤醒goroutine，并设置唤醒标志
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime.Semrelease(&m.sema)
                return
            }
            old = m.state
        }
    }
```

- Fast path，先通过去掉`AddInt32`去掉锁标志，同一时刻，Unlock的协程只可能有一个，这个操作一定成功。

  - `if (new+mutexLocked)&mutexLocked == 0`用来检验是不是对未加锁的Mutex做了Unlock操作
    - 如果state本来是无锁状态，对其`AddInt32`操作后，在执行if条件，结果是ture。
- 还需要”交代后事“，主要是指唤醒休眠的goroutine：

  - 如果这把锁没有等待者了，无需”交代“了。有唤醒的waiter或者锁已经又被抢到了，也无需”交代了“。

    - `if old>>mutexWaiterShift == 0`表示没有等待者。继续用int8来说明（后续的位运算举例都是用int8）
      - 假设有一个等待者的情况，old的值：00000100
      - 00000100>> 2 = 00000001，就通不过这条if，反之无等待者就能通过。
      - 后面都没人了，自然没事了直接return了。
    - `if old&(mutexLocked|mutexWoken) != 0`表示有唤醒的waiter或者锁已经又被抢到了。
      - mutexLocked | mutexWoken ---> 00000001 | 00000010 = 00000011
      - 通过了if至少说明，当前state至少满足mutexLocked或者mutexWoken任意一个
      - mutexLocked：锁已经又被抢到了，证明在竞争的活跃goroutine在当前Unlock的`AddInt32`方法后，立马拿到了锁，这里就是Lock方法中`插队`的情况，`插队`的goroutine会在自己的Unlock中做“交代”的。
      - mutexWoken：Unlock时已经有唤醒的waiter的情况在这里不会出现的场景。
        - 它会出现在下个版本中自旋的情况。
  - `new = (old - 1<<mutexWaiterShift) | mutexWoken`需要减去一个等待者数量，然后加上锁的唤醒状态，因为接下来就要唤醒一个了。这里可以回过头去看Lock中的awoke部分代码解释了。
  - 执行CAS，失败重试
    - 失败的情况发生在其他执行Lock的goroutine在修改唤醒状态和等待者数量。


### 问题

- 站在一个新goroutine的视角来看，它尝试去抢锁的时候，发现这会锁正在被持有，按照v2的逻辑，它应该被休眠。但是很可能极短的时间后（临界区代码执行很快），锁就空出来了，如果这时它能有机会再等一会的话，那它就有可能避免调休眠的命运从而提升性能。

## mutexV3

- 其他构造还是一样，但是Lock方法中加上了自旋

```go
func (m *Mutex) Lock() {
        // Fast path: 幸运之路，正好获取到锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }

        awoke := false
        iter := 0
        for { // 不管是新来的请求锁的goroutine, 还是被唤醒的goroutine，都不断尝试请求锁
            old := m.state // 先保存当前锁的状态
            new := old | mutexLocked // 新状态设置加锁标志
            if old&mutexLocked != 0 { // 锁还没被释放
                if runtime_canSpin(iter) { // 还可以自旋
                    if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                        atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                        awoke = true
                    }              
                    runtime_doSpin()
                    iter++
                    continue // 自旋，再次尝试请求锁
                }
                new = old + 1<<mutexWaiterShift
            }
            if awoke { // 唤醒状态
                if new&mutexWoken == 0 {
                    panic("sync: inconsistent mutex state")
                }
                new &^= mutexWoken // 新状态清除唤醒标记
            }
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                if old&mutexLocked == 0 { // 旧状态锁已释放，新状态成功持有了锁，直接返回
                    break
                }
                runtime_Semacquire(&m.sema) // 阻塞等待
                awoke = true // 被唤醒
                iter = 0
            }
        }
    }
```

- 在锁还被释放的情况下，加上了自旋

  ```go
   if old&mutexLocked != 0 { // 锁还没被释放
   	if runtime_canSpin(iter) { // 还可以自旋
         if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 && atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
             awoke = true        
          }              
          runtime_doSpin()
          iter++
          continue // 自旋，再次尝试请求锁                         
       }
       new = old + 1<<mutexWaiterShift
   }
  
  if awoke { // 唤醒状态
      if new&mutexWoken == 0 {
         panic("sync: inconsistent mutex state")
      }
      new &^= mutexWoken // 新状态清除唤醒标记            
  }
  ```

  - `if runtime_canSpin(iter)`判断能否自旋，取决于硬件条件和允许的自旋次数（iter）。
  - 先不考虑`awoke`的变化。临界区代码执行时间很短的情况下，spin多几次，可能就能跳过等待拿到锁。直到达到自旋限制后依然没抢到锁，才加入等待者队列。
  - 场景分析：
    - g1、g2、g3是三个goroutine
    - 现有g1正持有锁，g2在休眠排队
    - 这时g3来抢锁了，由于g1还未释放锁，g3理应去休眠
    - 但是g3在自旋“拖延时间”，等到g1释放后，g1的UnLock中别去唤醒g2，从而让自己获得更高的优先级。这该用什么方法去做呢?
    - UnLock直接return的条件是`if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken) != 0`
      - 对于g3的情况，`old>>mutexWaiterShift == 0`不符合
      - `mutexLocked`不一定能拿捏，UnLock刚刚才释放掉锁，这会可能有其他活跃的goroutine也在抢
      - 只能从`mutexWoken`入手了，如果g3在g1释放锁之前，就把`mutexWoken`状态给加上，那g2就不可能在这个阶段被唤醒了。g3的竞争对手只有活跃的goroutine，包括来的稍早正在自旋的，和新来直接抢锁的。
      - 不管谁抢到，都少了一次goroutine的休眠与唤醒
  - v2中，锁一旦加上了mutexWoken状态后，一定是唤醒了一个已休眠的goroutine。但是自旋模式时，新来的活跃goroutine为了多一点机会，所以它需要“骗”一下执行UnLock的goroutine，“骗”的方式是将锁加上mutexWoken状态来”冒充“被唤醒的等待者，让UnLock不要去唤醒等待者了。
  - 接下来的重点就是何时”行骗“，给锁加mutexWoken了，这个过程发生在自旋中。
    - `!awoke`首先在这个Unlock的goroutine时，已经awoke，就不需要了，这说明”行骗“成功了。
    - `old&mutexWoken == 0`，有了就没必要添加了。
    - `old>>mutexWaiterShift != 0`没有等待者情况下，不需要”冒充“。
    - `atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken)`CAS得成功。
  - 总结：
    - UnLock时，不需要”交代后事“的情况有三种
      - 没有等待者了
      - 已经被抢到锁了
      - 有活跃goroutine进来，虽然这会锁还未被释放，但是它在自旋`拖延时间`，想”骗“一下以便于插队，否则锁没被释放的情况下，它一定得被休眠。

### 问题

- 新来的goroutine靠“骗”给自己带来了优先级。如果这种现象一直出现，那被休眠的goroutine会永无出头之日，也就是进入了饥饿状态。v4引入了饥饿模式来做一个在性能和公平之间的平衡。



## mutexV4

- 跟之前的实现相比，当前的 Mutex 最重要的变化，就是增加饥饿模式。
  - 等待者等待的时间一旦超过1毫秒，锁就有可能进入饥饿模式。 
  - 正常模式下，等待者都是进入先入先出队列，被唤醒的等待者并不会直接持有锁，需要和新来的goroutine竞争。
    - 补充说明下，需要被唤醒的等待者在唤醒之前来抢锁的新goroutine，在v3中，可以通过“骗”让这个等待者不会被唤醒。
    - 而和唤醒之后来的进来的新goroutine之间，会相互竞争。
  - 被唤醒的等待者可能抢不到锁，这时，它又会被休眠，并被插入队列的前面，如果等待者获取不到锁的时间超过阈值1毫秒，那么进入饥饿模式。
  - 饥饿模式下，Mutex的现任拥有者使用完锁后，直接交给队列最前面的等待者。
  - 如果此等待者已经是队列中的最后一个了，就会转入正常模式。
  - 如果此等待者的等待时间小于1毫秒，也会转入正常模式。

### 新的const

```go
 const (
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken
        mutexStarving // 从state字段中分出一个饥饿标记
        mutexWaiterShift = iota
    
        starvationThresholdNs = 1e6
    )
```

- Mutex.state作为复合型字段，多了一个含义：mutexStarving，表示当前锁是否处于饥饿模式。
- starvationThresholdNs表示等待者等待多久会进入饥饿模式，这里值是1毫秒。



### Lock

```go
    func (m *Mutex) Lock() {
        // Fast path: 幸运之路，一下就获取到了锁
        if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
            return
        }
        // Slow path：缓慢之路，尝试自旋竞争或饥饿状态下饥饿goroutine竞争
        m.lockSlow()
    }
    
    func (m *Mutex) lockSlow() {
        var waitStartTime int64
        starving := false // 此goroutine的饥饿标记
        awoke := false // 唤醒标记
        iter := 0 // 自旋次数
        old := m.state // 当前的锁的状态
        for {
            // 锁是非饥饿状态，锁还没被释放，尝试自旋
            if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
                if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                    awoke = true
                }
                runtime_doSpin()
                iter++
                old = m.state // 再次获取锁的状态，之后会检查是否锁被释放了
                continue
            }
            new := old
            if old&mutexStarving == 0 {
                new |= mutexLocked // 非饥饿状态，加锁
            }
            if old&(mutexLocked|mutexStarving) != 0 {
                new += 1 << mutexWaiterShift // waiter数量加1
            }
            if starving && old&mutexLocked != 0 {
                new |= mutexStarving // 设置饥饿状态
            }
            if awoke {
                if new&mutexWoken == 0 {
                    throw("sync: inconsistent mutex state")
                }
                new &^= mutexWoken // 新状态清除唤醒标记
            }
            // 成功设置新状态
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                // 原来锁的状态已释放，并且不是饥饿状态，正常请求到了锁，返回
                if old&(mutexLocked|mutexStarving) == 0 {
                    break // locked the mutex with CAS
                }
                // 处理饥饿状态

                // 如果以前就在队列里面，加入到队列头
                queueLifo := waitStartTime != 0
                if waitStartTime == 0 {
                    waitStartTime = runtime_nanotime()
                }
                // 阻塞等待
                runtime_SemacquireMutex(&m.sema, queueLifo, 1)
                // 唤醒之后检查锁是否应该处于饥饿状态
                starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
                old = m.state
                // 如果锁已经处于饥饿状态，直接抢到锁，返回
                if old&mutexStarving != 0 {
                    if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
                        throw("sync: inconsistent mutex state")
                    }
                    // 有点绕，加锁并且将waiter数减1
                    delta := int32(mutexLocked - 1<<mutexWaiterShift)
                    if !starving || old>>mutexWaiterShift == 1 {
                        delta -= mutexStarving // 最后一个waiter或者已经不饥饿了，清除饥饿标记
                    }
                    atomic.AddInt32(&m.state, delta)
                    break
                }
                awoke = true
                iter = 0
            } else {
                old = m.state
            }
        }
    }
    
    
```

lockSlow：

- `if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) `进入自旋
  - 只有在非饥饿模式下，并且锁没释放才能自旋。
  - 自旋中，“行骗”的方式和v3一样。

- 构造用于替换旧state的新值

  - `if old&mutexStarving == 0`非饥饿模式下，给构造的新值加锁。

    - 反之饥饿模式下，有比当前goroutine更应该拿到锁的goroutine，所以是不能加锁持有状态的。

  - `if old&(mutexLocked|mutexStarving) != 0`如果锁又已经被持有了（其它goroutine抢到了），或者已经进入饥饿模式了。当前goroutine都只能去休眠排队。

  - ```go
    if starving && old&mutexLocked != 0 { 
        new |= mutexStarving // 设置饥饿状态 
    }
    ```

    - 如果此goroutine已经处在饥饿状态（starving），并且锁还被持有，那么，我们需要把此 Mutex 设置为饥饿模式。
    - starving的值由当前goroutine休眠时间决定

- `if awoke`清楚唤醒标志逻辑和v3一样
- CAS操作，成功之后，需要处理锁的状态。
  - 这里的替换成功，不一定是带着持有锁状态。
  - 如果替换前的old，即没有持有锁，也不是饥饿模式，那可以直接抢锁成功，然后退出了。
    - 分析一下，在这个位置，不同的old值情况下，new值是什么：
      - 非饥饿模式 + 未被持有：new值新增了锁状态，等待者数量没变
      - 非饥饿模式 + 已被持有 ：new值给等待者+1
      - 饥饿模式 + 未被持有：new值给等待者+1
      - 饥饿模式 + 已被持有：new值给等待者+1
  - 除去第一种情况，直接拿到了锁，跳出Lock了，去其余情况都得考虑休眠等待的事情：
    - 当前goroutine休眠前，先保存下当前时间，用于唤醒后计算自己被休眠了多久。
    - 把自己放在休眠队列中
      - `queueLifo := waitStartTime != 0`queueLifo能判断该Lock的goroutine是否是首次被休眠，如果是首次就排队到队列的尾部。
      - 如果不是首次，证明曾经被唤醒过，不过没抢到，所以它排在队首，让自己在下一次唤醒中，优先级变高。
      - 注：`runtime_SemacquireMutex(&m.sema, queueLifo, 1)`这里的休眠方法可以指定让自己进入休眠队列的什么位置。
    - 休眠醒来后，发现锁已经是饥饿模式了`if old&mutexStarving != 0 `
      - 自己直接拿锁，设置锁状态，并且给等待者数量-1
      - `old>>mutexWaiterShift == 1`如果自己已经是最后一个等待者了，需要为锁去除掉饥饿状态
      - `!starving`如果等待时间小于一毫秒，也需要为锁去除掉饥饿模式。
    - 醒来之后，锁不是饥饿模式，则重新去抢锁
      - 重新抢锁时，由于锁不是饥饿模式，所以在构造新值时能带上锁持有标志
      - 如果starving的值是true，重新抢锁时会将锁设置为饥饿模式。这样即使在CAS交换后，依然没抢到锁，进入了休眠，那它被唤醒后，一定能马上获得锁。

### Unlock

```go
func (m *Mutex) Unlock() {
        // Fast path: drop lock bit.
        new := atomic.AddInt32(&m.state, -mutexLocked)
        if new != 0 {
            m.unlockSlow(new)
        }
    }
    
    func (m *Mutex) unlockSlow(new int32) {
        if (new+mutexLocked)&mutexLocked == 0 {
            throw("sync: unlock of unlocked mutex")
        }
        if new&mutexStarving == 0 {
            old := new
            for {
                if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                    return
                }
                new = (old - 1<<mutexWaiterShift) | mutexWoken
                if atomic.CompareAndSwapInt32(&m.state, old, new) {
                    runtime_Semrelease(&m.sema, false, 1)
                    return
                }
                old = m.state
            }
        } else {
            runtime_Semrelease(&m.sema, true, 1)
        }
    }
```

- UnLock，“交代后事”的逻辑变化的核心点在于，饥饿模式下直接唤醒休眠队列中的第一个。
- 非饥饿模式下
  - for循环过程中，锁可能切换成了饥饿模式
  - 这种情况也不需要“交代后事”了
  - 饥饿模式一定是正在Lock的goroutine中设置的，它自己会去处理。
    - 饥饿模式下，新来的goroutine，不能自旋，也拿不到锁，都会去排队
    - 把锁从正常模式切换到的饥饿模式的goroutine一定是休眠过并唤醒的
      - Lock中starving取值逻辑可见
    - 在切换到饥饿模式之前，新来了的goroutine，它们之间会相互竞争。但是即使starving==true的这个goroutine竞争失败了，那它CAS成功后，再次休眠唤醒后，也能直接拿到锁。



## Mutex中的位运算逻辑

从v2版本开始就用上了位运算，Mutex中得State字段类型是int32，它又被分为了三个部分：

 ![](/v1Mutex位运算.png)

- mutexWaiters表示锁的等待者数量、mutexWoken表示正在抢锁的goroutine是否有被唤醒的、mutexLocked表示锁是否被持有
- 锁的整个状态变化就是这三个部分变化的组合，那问题就是：当需要修改某一部分值的时候，怎样去避免影响到其他部分？自然就是通过位运算。

程序中的所有数在计算机内存中都是以二进制的形式储存和运算的。位运算就是直接对整数在内存中的二进制位进行操作。比如，and运算本来是一个用于布尔类型的逻辑运算符，但整数与整数之间也可以进行and运算。举个例子，6的二进制是110，11的二进制是1011，那么6 and 11的结果过程：

- 将6的二进制数补成11的二进制数的长度，也就是0110
- 0表示False，1表示True
- 0110 and 1011 = 0010，转为十进制，就是2。

在go中也有位运算符：

| 运算符 | 名称     | 说明                                           | 示例                                                 |
| ------ | -------- | ---------------------------------------------- | ---------------------------------------------------- |
| <<     | 左移     |                                                | 1 << 1 (1换二进制等于1，左移一位等于10，转十进制为2) |
| >>     | 右移     |                                                | 2 >> 1 (2换二进制等于10，右移一位等于1，转十进制为1) |
| &      | 按位与   |                                                | 6 & 11 = 2，过程见上                                 |
| \|     | 按位或   |                                                | 6 \| 11 ---> 0110 \| 1011  = 1111 ---> 15            |
| ^      | 按位异或 | 相同就1，不同就是0                             | 6 ^ 11  ---> 0110 ^ 1011 = 0010 ---> 2               |
| &^     |          | 以运算符左边数据为准，相异的位保留，相同位清零 | 1 &^ 0 = 1、1 &^ 1 = 0、0 &^ 0 = 0、0 &^ 1 = 0       |

为了对Mutex.State做位运算，定义了三个const，用于位运算右边的数据

- 三个const

  - mutexLocked =  00000000 00000000 00000000 00000001
  - mutexWoken = 00000000 00000000 00000000 00000010
  - mutexWaiterShift = 00000000 00000000 00000000 00000010

- 为了便于解释，采用int8来代替int32，同样最低1位表示锁是否被持有，倒数第二地位表示是否有唤醒的goroutine，其余位表示等待者数量

- 三个const变成

  - mutexLocked = 00000001
  - mutexWoken = 00000010
  - mutexWaiterShift = 00000010

- 实际操作

  - 锁是否被持有

    改变锁是否被持状态有只需要对mutexLocked部分操作

    - 加上锁标志
      - 不能纯粹对state+1操作，因为已持有锁的情况下会进位，影响到了mutexWoken部分
      - 采用key对mutexLocked做 | 操作
        - 有锁状态：11111101 | 00000001 = 11111101
        - 无锁状态：11111100 | 00000001 = 11111101
    - 判断是否持有锁
      - 对mutexLocked做 & 操作
      - 有锁状态：11111101 & 00000001 = 000000001 
      - 无锁状态：11111100 & 00000001 = 000000000
      - key & mutexLocked != 0 就证明原本是持有锁的
    - 解除持有锁状态
      - 理论上同样不能简单使用state-1
      - 采用&^运算，保留相异位，去除相同位
        - 11111101 &^ 00000001 = 11111100
      - 但是在Mutex的场景中，只有可能是持有锁的goroutine做解除操作，所以不存在对无锁状态做解锁操作，直接使用-1操作就好了
    
  - 是否有被唤醒的goroutine
  
    - 和`锁是否被持有`一样,只是右边的数据变成了const mutexWoken
    - | 用于添加唤醒状态
    - & 用于判断是否是唤醒状态
    - &^ 用于去除
  
  - 等待者数量
  
    - 对等待者数量的操作只有+1和-1
    - 增加等待者数量，就是针对倒数第三位+1
      - const mutexWaiters的值等于2，将1左移两位，得到 00000100，拿State和它做+运算即可
      - 增加：state + 1 << mutexWaiterShift  ---> 00000001 + 00000100 = 00000101
    - 减少等待者数量类似
      - state - 1<<mutexWaiterShift 