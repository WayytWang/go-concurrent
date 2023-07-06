值类型、引用类型、值传递、引用传递

var、new、make

懵不懵



- 常量的值。
- 基本类型值的字面量。
- 算术操作的结果值。
- 对各种字面量的索引表达式和切片表达式的结果值。不过有一个例外，对切片字面量的索引结果值却是可寻址的。
- 对字符串变量的索引表达式和切片表达式的结果值。
- 对字典变量的索引表达式的结果值。
- 函数字面量和方法字面量，以及对它们的调用表达式的结果值。
- 结构体字面量的字段值，也就是对结构体字面量的选择表达式的结果值。
- 类型转换表达式的结果值。
- 类型断言表达式的结果值。
- 接收表达式的结果值。



go的数据类型

```go
int
float
string

slice
map

struct
chan


```



基本类型：int、float、string、bool

聚合类型：array、struct

引用类型：slice、map、pointer、func、chan

接口类型：interface



类型

是否并发安全



从值类型和引用类型开始

```go
// slice
var s []int
log.Println(s == nil)

// map
var m map[int]int
log.Println(m == nil)

// function
var f func(int) int
log.Println(f == nil)

// channel
var c chan int
log.Println(c == nil)

// pointer
var p *int
log.Println(p == nil)
```



引用类型初始值都是nil，都引用了某个类型

- 切片基于数组实现的
- map基于数组实现的
- 函数需要堆栈
- channel
- 指针是指向某个类型



```go
var i int
log.Println(&i)

var s []int
log.Println(s == nil)
```

var：声明一个变量

对于int类型的i，会发现i

