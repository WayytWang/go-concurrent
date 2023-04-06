---
typora-root-url: pic
---

# pflag包

github地址：https://github.com/spf13/pflag

`pflag`包是用于做命令行的参数解析，相较于标准库的`flag`，它的功能更为强大，`kubectl`就是使用的该库做命令行解析



## FlagSet

所有的flag，都被`FlagSet`管理支持

通过`NewFlagSet`可以创建

```go
var version bool
flagSet := pflag.NewFlagSet("test", pflag.ContinueOnError)
flagSet.BoolVar(&version, "version", true, "Print version information and quit.")
```

- ErrorHandling

  ```go
  const (
  	// ContinueOnError will return an err from Parse() if an error is found
  	ContinueOnError ErrorHandling = iota
  	// ExitOnError will call os.Exit(2) if an error is found when parsing
  	ExitOnError
  	// PanicOnError will panic() if an error is found when parsing flags
  	PanicOnError
  )
  ```

一般不需要自创建`FlagSet`，可以直接使用`pflag`包内定义好的`CommandLine`

CommandLine

  ```go
  // CommandLine is the default set of command-line flags, parsed from os.Args.
  var CommandLine = NewFlagSet(os.Args[0], ExitOnError)
  ```

  

## 参数解析

### 非选项参数

```go
pflag.Parse()

fmt.Printf("参数数量: %d\n", pflag.NArg())
fmt.Printf("参数列表: %v\n", pflag.Args())
fmt.Printf("第2个参数: %s\n", pflag.Arg(1))
```

 ![](/pflag_args_e.png)



### 选项参数及缩写

```go
// wang是默认值
pflag.String("name", "wang", "Input Your Name")
// IP类型（flag包不支持类型） P结尾的函数多一个shorthand参数
pflag.IPP("ipaddress", "i", []byte("127.0.0.1"), "Input Your IP")
pflag.Parse()

name, _ := pflag.CommandLine.GetString("name")
fmt.Printf("name: %s \n", name)
ip, _ := pflag.CommandLine.GetIP("ipaddress")
fmt.Printf("ip: %s \n", ip.String())
```

 ![](/pflag_name_e.png)



### 绑定变量

```go
var name string
pflag.StringVarP(&name, "name", "n", "wang", "Input Your Name")
pflag.Parse()

fmt.Printf("name:%s \n", name)
```

就不需要通过`FlagSet`的`GETXXX`方法获取值了



### 废弃选项

版本迭代的时候，会废弃一些选项，但是希望用户用到老的选项的时候，能给个提示

```go
pflag.String("uname", "wang", "Input Your Name")
pflag.String("name", "wang", "Input Your Name")
// name选项被废弃掉
pflag.CommandLine.MarkDeprecated("name", "please use --uname instead")
pflag.Parse()
```

 ![](/plag_deprecated_e.png)



### 保留选项但弃用简写

```go
pflag.StringP("name", "n", "wang", "Input Your Name")
pflag.CommandLine.MarkShorthandDeprecated("name", "please use --name only")
pflag.Parse()
```

 ![](/pflag_shorthandDeprecated_e.png)

