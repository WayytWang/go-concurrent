项目中往往都有配置文件，推荐的库为:`viper`

github地址： https://github.com/spf13/viper



## 配置写入Viper

### 设置默认值

```go
viper.SetDefault("loglevel", "debug")
fmt.Println(viper.GetString("loglevel"))
```



### 读取配置文件

```go
// viper会从添加的path中依次找到配置文件
viper.AddConfigPath(".")
viper.AddConfigPath("../")
// SetConfigName可不带拓展名
//viper.SetConfigName("config")

// SetConfigFile需要带上扩展名
viper.SetConfigFile("config.yaml")

// 将配置读到viper中
if err := viper.ReadInConfig(); err != nil {
	panic(err)
}

fmt.Printf("Used configuration file is:%s \n", viper.ConfigFileUsed())
```



### 配置热加载

```go
viper.SetConfigFile("./config.yaml")
if err := viper.ReadInConfig(); err != nil {
	panic(err)
}

go func() {
	viper.WatchConfig()
	// 当config.yaml发生变化时，回调函数会被执行
	viper.OnConfigChange(func(e fsnotify.Event) {
		fmt.Println("Config file changed:", e.Name)
	})
}()
select {}
```

### 设置配置值

```go
type server struct {
	Address string
	LogPath string `mapstructure:"Log_path"`
}
type config struct {
	Server  server
}

viper.SetConfigFile("./config.yaml")
if err := viper.ReadInConfig(); err != nil {
	panic(err)
}
// 会覆盖从配置文件获取的值
viper.Set("server.address", "9999")

var conf config
err := viper.Unmarshal(&conf)
if err != nil {
	panic(err)
}
fmt.Printf("conf:%+v", conf)
```

### 命令行参数

配合pflag
```go
pflag.String("name", "wang", "Input Your Name.")
pflag.String("code", "go", "Input Your Code.")
viper.BindPFlags(pflag.CommandLine)

fmt.Println(viper.AllKeys())
```

### 环境变量

```go
os.Setenv("VIPER_USER_ID", "user_id_123")
os.Setenv("VIPER_USER_key", "user_key_123")

// 读取环境变量
viper.AutomaticEnv()
// 设置写入viper的环境变量前缀
viper.SetEnvPrefix("VIPER")
viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_", "-", "_"))
// 绑定环境变量到key
viper.BindEnv("user.id")
viper.BindEnv("user.key")

fmt.Println(viper.GetString("user.id"))
fmt.Println(viper.GetString("user.key"))
```





## 从Viper读取配置

Getxxx

```go
viper.SetConfigFile("./config.yaml")
if err := viper.ReadInConfig(); err != nil {
	panic(err)
}

address := viper.GetString("server.address")
fmt.Println(address)
```

反序列化

```go
// tag key mapstructure
type server struct {
	Address string
	LogPath string `mapstructure:"Log_path"`
}
type appAuthInfo struct {
	Id     string
	Secret string
}
type config struct {
	Server  server
	AppAuth []appAuthInfo
}
	
viper.SetConfigFile("./config.yaml")
if err := viper.ReadInConfig(); err != nil {
	panic(err)
}

var conf config
err := viper.Unmarshal(&conf)
if err != nil {
	panic(err)
}
fmt.Printf("conf:%+v", conf)
```

