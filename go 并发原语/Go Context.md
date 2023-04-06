



gin的Context

```
// Context is the most important part of gin. It allows us to pass variables between middleware,
// manage the flow, validate the JSON of a request and render a JSON response for example.
```



标准库部分功能

`context`

gin.Context实现了标准库的`Context`接口

实现方式是 gin.Context.





核心点

这是一段`database/sql`标准库中，最终执行sql语句的代码

![](D:\note\note\wang\go\go 并发原语\pic\sql_context.png)

- sql往往是比较耗时的，在它执行前，先判断还有没有必要执行
- 如果它的调用方认为没有必要执行sql了，选择不浪费资源



```go
done := make(chan struct{})

go func() {
	time.Sleep(3 * time.Second)
    close(done)
}()	

select {
	case <-done:
		break
}
fmt.Println("done")		
```

- `done`是一个无缓冲的管道
- select语句没有默认分支的时候，当没有case被选中，是会阻塞住的
- 当对一个被关闭的channel，接收操作会接收到channel元素类型的零值
- 所以在goroutine中，三秒后done被关闭，此刻select不再阻塞



另一个作用做超时处理

```go
ctx, cancel := context.WithCancel(context.Background())
go func() {
	time.Sleep(3 * time.Second)
	cancel()
}()

go func() {
	time.Sleep(5 * time.Second)
	fmt.Println("goroutine done")
}()

select {
case <-ctx.Done():
	fmt.Println("goroutine timeout")
}
```

- 假设第二个`go`语句开启的协程，是调用其他服务，但可能会特别耗时
- 而这个调用方只有3秒钟时间，如果太长就放弃等待调用结果了
- 这时也可以使用context



https://www.ardanlabs.com/blog/2019/09/context-package-semantics-in-go.html


Incoming requests to a server should create a Context

Outgoing calls to servers should accept a Context

Do not store Contexts inside a struct type; instead, pass a Context explicitly to each function that needs it

The chain of function calls between them must propagate the Context

Replace a Context using WithCancel,WithDeadline,WithTimeout,or WithValue

When a Context is canceled,all Contexts derived from it are also canceled

The same Context may be passed to functions running in different goroutines;Contexts are safe for silmultaneous use by multiple goroutines

Do not pass a nil Context,even if a function permits it.Pass a TODO context if you are unsure about which Context to use

Use context values only for request-scoped data that transits processes and APIs, not for passing optional parameters to functions
