# Mutex的实现

- 虽然go鼓励用通信的方式来共享内存，但是用共享内存的方式来通信依然是不可或缺的。
- 比如某一个变量或者某一段代码，可能会被多个协程同时执行，但是我又只希望同一时刻只有一个协程能操作，就需要用到互斥锁。这个变量或者这段代码称为临界区。每一个协程在执行临界区代码前，都需要使用互斥锁的Lock方法尝试抢执行权，如果抢不到会阻塞住，直到抢到了才能往后执行。当执行完后需要使用互斥锁的UnLock方法将执行权交出去，以便于其他协程能抢到执行权。
- 本文的逻辑是通过一步步实现互斥锁的方式，来分析go标准库的Mutex的原理。从而可以尽可能避免在使用过程中踩到莫名奇妙的坑。



在开始实现互斥锁时，先要了解CAS

## CAS

- CAS(compare and swap)意思是比较并交换。

- 以go标准库中的atomic.CompareAndSwapInt32方法为例

  ```go
  // CompareAndSwapInt32 executes the compare-and-swap operation for an int32 value.
  func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
  ```

  - 它接收三个入参，一个指针addr，一个旧值old，一个新值new
  - 这个操作分为三步:
    - 首先取addr指向的值
    - 然后与old比较，如果不相等，返回false，操作失败。
    - 最后将addr指向的值改成new
  - 关键在于，机器层面保证了这三步的原子性，既执行过程中不会被其他协程所打断。
    - 普通代码可能在执行过程中突然被打断，cpu转而去执行其他协程了。

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
  - g2的Lock方法成功。

#### 问题

- 虽然实现了需求，但是毫无性能可言。g1什么正事也干不了，但是一直吃着cpu的资源。

  

## mutexV1

- 这版本的目的，就是设法让拿不到锁的goroutine休眠，别在浪费资源了。

### mutex结构

- key
  - 它的值不在是0和1,
  - 0表示锁未被持有，1表示锁已经被持有，n表示锁被持有，n-1表示锁的等待者
- sema
  - 信号量
  - 信号量用于将当前goroutine的休眠已经唤醒

### 信号量的使用

- semacquire
- semrelease



Lock方法和Unlock方法不在是对key做0和1之间的替换了，而是采用+1和-1操作

这样不仅能记录锁的持有状态，还能记录排队的goroutine数

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

- 这里也有for循环，for循环是因为有多个goroutine同时在竞争CAS操作

  - 从`v := *val`到CAS之间可不是原子操作，有可能刚刚拿到的v=1，等到执行到CAS这一行时，被其他执行这段逻辑的协程改成2了。所以需要for循环去保证CAS成功。
  
- 但是和v0版本的for循环有很大区别

  - v0的目的是0和1之间做交换，抢锁时从0到1，for循环不仅在一起抢锁的协程在竞争，还在等着已经有锁的协程释放锁。

  

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

- xadd执行完后，key值如果等于1，那证明当前goroutine已经拿到了锁，可以直接返回执行后续临界区代码了
- 如果key值大于1，那证明已经拿到了锁的待使用资格，不过还有其他协程也拿到了待使用资格，得去排队。
- 这里排队的方式是将自己休眠，而不是v0那样疯狂循环消耗CPU资源

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
- 如果没有其他goroutine在Lock，这是xadd方法后，key就变成0，这是不用管了直接返回就好了
- 但是执行临界区代码的这段日子里，可能有很多goroutine在排队。他们会对key做+1的操作
- 所以这里key可能大于0，证明有一些goroutine在排队，那些排队的goroutine，这会休眠需要被唤醒，所以Unlock这不能偷偷的返回了

### 问题

-  v1版本中，拿到锁的goroutine，除非一开始直接获取成功，其他的都会经过休眠
- 休眠再唤醒，是涉及到协程的切换，这也是有损性能的
- 能不能在Unlock解锁后，有发现正在请求锁的活跃goroutine的话，就让它先来，这就少了一次休眠再唤醒的过程，有助于提升性能



  

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
  - 由key变成了state
  - state的类型是int32，由32个bit组成
  - 32bit的state可分为三个部分 mutexWaiters|mutexWoken|mutexLocked，所以state是一个复合型字段
    - mutexWaiters表示锁的等待者数量
    - mutexWoken表示锁是否有唤醒的goroutine
    - mutexLocked表示锁是否被持有
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

- 最好的情况，原state等于0，然后通过CAS将state的mutexLocked部分加上，成功直接返回。

- 其他情况就需要去竞争了

  - `new := old | mutexLocked `通过位运算将state加上锁状态

  - `if old&mutexLocked != 0`说明原本就是加过锁状态的，需要给等待者加上1

    - `new = old + 1<<mutexWaiterShift`

  - awoke的判断先不要看

  - 尝试CAS

    - 成功

      - 再次判断`if old&mutexLocked == 0`。此时发现加锁状态没了，这时不需要像v1那样休眠，而是直接执行了，即使当前goroutine不是排队的第一个。

        这是区别v1的主要逻辑。

      - 如果此时锁的状态还是有，那说明当前goroutine不可避免还是得去休眠，同时将awoke改为true，标志着当前goroutine休眠后再次被唤起后已经是一个被唤醒过的goroutine了。

    - 失败

      - 没啥好说的，继续取值，继续构造旧值，继续竞争

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

- 先通过去掉锁标志，同时间一把锁Unlock的只可能有一个

- 如果这个锁原本就是没加锁的状态，就直接给panic了。

- v1版本中Unlock后不能直接离开，v2也一样需要善后

  - 如果这把锁没有等待者了，其实就一个goroutine用，`if old>>mutexWaiterShift == 0`就不用管了
  - `if old&(mutexLocked|mutexWoken) != 0`
    - mutexLocked表示加上锁了
      - 第一次循环不会出现这种情况，后面的可能来了新的goroutine拿到了锁，导致原子操作失败
    - mutexWoken表示有唤醒的goroutine了
      - ?不知道何时会出现，唤醒状态是Unlock中才会出现的结果，之后的cas不成功的话，应该不会出现这个值
    - `old&(mutexLocked|mutexWoken)`来判断目前的state是不是有唤醒的waiter或者已经加上锁了
    - 有唤醒的waiter，或者已经加上锁了，说明至少还有活跃着的goroutine在操作这把锁，他们会自己在Unlock的中善后
  - 但是如果有等待者并且没有被唤醒的waiter，那这里就需要唤醒一个
    - 等待者－1
    - 唤醒标志mutexWoken设置好
      - mutexWoken状态设置好后，会去唤醒一个在Lock中休眠的goroutine，被唤醒后它会取消掉这个标志
    - 尝试CAS，不行再来
  
  

### 问题

- 站在一个新goroutine的视角来看，它尝试去抢锁的时候，发现这会锁正在被持有，按照v2的逻辑，它应该被休眠。但是很可能极短的时间后，锁就空出来了，如果这时它能有机会再等一会的话，那它就有可能避免调休眠的命运从而提升性能。就好像挂科了，如果能给次补考，就比来年和学弟学妹们一起考要爽很多。

## mutexV3

- 用上了spin，自旋

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
                    !awoke 不是被唤醒状态
                    old&mutexWoken==0 : 旧状态锁被释放了
                    old>>mutexWaiterShift != 0 ： 有等待者
                    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken)  CAS抢锁 （不排队）
                    
                    
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



## mutexV4

- 每次被唤醒的goroutine，在抢锁失败后，他又去了等待队列的尾部

```go

   type Mutex struct {
        state int32
        sema  uint32
    }
    
    const (
        mutexLocked = 1 << iota // mutex is locked
        mutexWoken
        mutexStarving // 从state字段中分出一个饥饿标记
        mutexWaiterShift = iota
    
        starvationThresholdNs = 1e6
    )
    
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

我们从请求锁（lockSlow）的逻辑看起。第 9 行对 state 字段又分出了一位，用来标记锁是否处于饥饿状态。

现在一个 state 的字段被划分成了阻塞等待的 waiter 数量、饥饿标记、唤醒标记和持有锁的标记四个部分。

第 25 行记录此 goroutine 请求锁的初始时间，第 26 行标记是否处于饥饿状态，第 27 行标记是否是唤醒的，第 28 行记录 spin 的次数。

第 31 行到第 40 行和以前的逻辑类似，只不过加了一个不能是饥饿状态的逻辑。它会对正常状态抢夺锁的 goroutine 尝试 spin，和以前的目的一样，就是在临界区耗时很短的情况下提高性能。

第 42 行到第 44 行，非饥饿状态下抢锁。怎么抢？就是要把 state 的锁的那一位，置为加锁状态，后续 CAS 如果成功就可能获取到了锁。

第 46 行到第 48 行，如果锁已经被持有或者锁处于饥饿状态，我们最好的归宿就是等待，所以 waiter 的数量加 1。

第 49 行到第 51 行，如果此 goroutine 已经处在饥饿状态，并且锁还被持有，那么，我们需要把此 Mutex 设置为饥饿状态。

第 52 行到第 57 行，是清除 mutexWoken 标记，因为不管是获得了锁还是进入休眠，我们都需要清除 mutexWoken 标记。

第 59 行就是尝试使用 CAS 设置 state。如果成功，第 61 行到第 63 行是检查原来的锁的状态是未加锁状态，并且也不是饥饿状态的话就成功获取了锁，返回。

第 67 行判断是否第一次加入到 waiter 队列。到这里，你应该就能明白第 25 行为什么不对 waitStartTime 进行初始化了，我们需要利用它在这里进行条件判断。第 72 行将此 waiter 加入到队列，如果是首次，加入到队尾，先进先出。如果不是首次，那么加入到队首，这样等待最久的 goroutine 优先能够获取到锁。此 goroutine 会进行休眠。

第 74 行判断此 goroutine 是否处于饥饿状态。注意，执行这一句的时候，它已经被唤醒了。第 77 行到第 88 行是对锁处于饥饿状态下的一些处理。

第 82 行设置一个标志，这个标志稍后会用来加锁，而且还会将 waiter 数减 1。

第 84 行，设置标志，在没有其它的 waiter 或者此 goroutine 等待还没超过1 毫秒，则会将 Mutex 转为正常状态。

第 86 行则是将这个标识应用到 state 字段上。





## RWMutex

- readerCount
- readerWait

### RLock

- readerCount + 1
  - 大于0 成功 
  - 小于0 有writer在等待 休眠

### RUnlock

- readerCount -1 
  - 大于0 结束
  - 小于0 rUnlockSlow
    - 判断是否需要唤醒一个writer readerWait-1 == 0

### Lock

- writer获得了内部的互斥锁
- readerCount - rwmutexMaxReaders readers可以知道有writer竞争了
- 当前有reader持有锁，需要等待，readerWait=readerCount

### Unlock

- 告诉reader没有活跃的writer readerCount + rwmutexMaxReaders 

- 唤醒阻塞住的readers
- writer释放锁
