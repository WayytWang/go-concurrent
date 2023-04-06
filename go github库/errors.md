---
typora-root-url: pic
---



github地址：https://github.com/pkg/errors

go项目中，避免对error重复处理，需要遵循一套error处理的原则，而实现这一套处理机制，需要用到`github.com/pkg/errors`库

## New

相较于标准里的errors.New()，该包New的errors，包含了堆栈信息

```go
package main

import (
	"fmt"
	"github.com/pkg/errors"
)

func main() {
	err := func2()
    // %+v可以打印堆栈
	fmt.Printf("%+v", err)
}

func func2() error {
	return func1()
}

func func1() error {
	return errors.New("test stack")
}
```

 ![](/errors_new_e.png)

## error Wrap

### Wrap

对于其它地方返回的error，Wrap后再往外抛，可携带堆栈信息。切记不要重复Wrap，否则打印控制台是灾难。

```go
package main

import (
	"fmt"
	"github.com/pkg/errors"
	"os"
)

func main() {
	err := func2()
	fmt.Printf("%+v", err)
}

// 不要重复Wrap了
func func2() error {
	return func1()
}

func func1() error {
	_, err := os.Open("./noExist")
    // 不Warp的话，main中打印的error就会丢失堆栈
	return errors.Wrap(err, "func1 open file error")
}
```

### Cause

被Wrap的error，可以通过Cause方法取出来根因

```go
err := func2()
causeErr := errors.Cause(err)
```

### WithMessage

只带信息，不带堆栈

```go
_, err := os.Open("./noExist")
return errors.WithMessage(err, "func1 open file error")
```

### WithStack

只带堆栈，不带信息

```go
_, err := os.Open("./noExist")
return errors.WithStack(err)
```

## error处理

### Is

配合对哨兵error的Wrap

```go
package main

import (
	"fmt"
	"github.com/pkg/errors"
	"io"
)

func main() {
	err := func2()
	fmt.Println(err == io.EOF) // false
	fmt.Println(errors.Is(err, io.EOF)) // true
}

func func2() error {
	return func1()
}

func func1() error {
	err := io.EOF
	return errors.Wrap(err, "fun1 error")
}
```



### As

配合实现error接口

```go
package main

import (
	"fmt"
	"github.com/pkg/errors"
	"strconv"
)

type MyError struct {
	Err   error
	Level int
}

func main() {
	err := func2()
    // *MyError实现了error
	var e *MyError
    // &e要
	fmt.Println(errors.As(err, &e)) // true
}

func (e *MyError) Error() string {
	return e.Err.Error() + " and error level:" + strconv.Itoa(e.Level)
}

func func2() error {
	return func1()
}

func func1() error {
	// mock 模拟从调用其他函数返回了error
	err := &MyError{
		Err:   errors.New("occur an error"),
		Level: 1,
	}
	return errors.Wrap(err, "func1 error")
}
```



