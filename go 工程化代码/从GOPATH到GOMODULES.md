# 从GOPATH到GOMODULES

- go包管理经历了坎坷的历程，不过当我开始学习go时，Go Modules已经很运用的很广泛了。所以我没有经历过go dep、go vendor等等这些，直接从学习阶段使用的GOPATH模式到了工作中Go Modules。
- 刚开始使用Go Modules时，go mod init + go mod tidy两个命令好像就足够了，对于相关的概念都是模糊的。
- 好多工具在开发工程中像空气一样重要，也像空气一样透明，似乎根本不需要了解的太深，最基础的几个命令怼上去就能解决大部分工作的问题了，比如git和Go Modules。当花一点时间仔细学习它们的设计是非常巧妙的。本文打算记录GOPATH和Go Modules的一些相关的知识点来加深我对它们的印象。

## GOPATH模式

安装go语言有三个很重要的环境变量：

| GOROOT           | GOBIN                          | GOPATH   |
| ---------------- | ------------------------------ | -------- |
| Go语言的安装路径 | Go程序生成的可执行文件保存路径 | 工作空间 |

GOROOT是安装go语言的路径，没什么特殊的。

关键是GOPATH，它的配置值是目录，可以配置多个目录，每个目录做为go语言的工作区间（workspace）。在没有开启gomod的情况下，go语言代码只能存在于GOPATH下，而GOPATH也有自己的独特的代码组织结构。

### 几个重要的命令

- 在分析GOPATH代码组织方式之前，先分析下几个重要的命令，分别是：
  - go build
  - go install
  - go get

#### go build

- 构建代码
- 如果构建的是含有main方法的代码，那么会得到一个可执行文件，这个执行文件出现在源码文件所在的目录下。
- 不含main方法的代码被执行，那肯定是直接或间接的被main方法调用到了。可执行文件是通过链接库文件来调用引入的代码的。而对不含main方法代码的构建，就是得到链接库文件，这些是以“.a”为扩展名的文件，这是链接库文件的归档文件。同一源码文件构建出来的归档文件和它所处的不同操作系统而有所不同。
- 构建得到的归档文件会存放在一个临时目录。

#### go install

- 安装代码
- 安装代码前，首先会构建代码，也就是说 go install = go build + 其他操作。
- 这里的其他操作指的是链接操作
  - 对含有main方法的代码构建结果，会移动到GOBIN目录下，如果没有GOBIN配置的话，会被搬到所在的GOPATH下的bin目录下。
  - 对于不含main方法的代码构建结果，会被搬到GOPATH下的pkg目录下。

#### go get

- 下载代码
- 会自动从主流的公用代码仓库，比如Github，并且执行go install的操作。
- 可见这三个命令是“层层加码“的。



### GOPATH

- GOPATH可以配置很多个。
- 在对命令的分析中提到了GOPATH/bin，GOPATH/pkg两个目录，这都属于GOPATH组织代码方式的范畴。
- 每一个GOPATH目录下，有三个子目录
  - src
    - 用来存放源码的。在自己写代码时，import的代码路径的父目录都是某一个GOPATH下的src。
  - pkg
  - bin

以go get github.com/gin-gonic/gin 的结果来看，来说明go代码的组织方式。

注意：此种方式必须是在关闭了Go Modules情况下拉取的代码才能体现的，也就是go env中GO111MODULE=off的情况。而开启了Go Modules下的情况在Go Modules的部分在做分析。

#### src

- src目录中存放着代码
- 代码存放在src/github.com/gin-gonic/gin下，目录层级会遵守go get的路径（按/符号分割）

#### pkg

- go install 和 go get的构建过程中所产生的归档文件都会放在这，目录层级和src一下。
- 之前提到过归档文件和操作系统会有关系，所以这里相对src目录会多一层操作系统平台目录，比如linux下pkg下gin文件的层级会是：pkg/linux_amd64/github.com/gin-gonic/gin

#### bin

- 在没有配置GOBIN的情况下，go install 和 go get的构建过程中所产生的可执行文件会在这里。



### GOPATH最致命的缺点

- 包的版本管理不方便。

- 如果A项目使用了gin的v1版本，B项目使用了gin的v2版本。
- 那么就只能给A,B两个项目创建两个GOPATH，并且将所需要版本的gin代码放在自己的GOPATH下。

  

## Go Modules

- Go1.11版本推出了Go Modules，所以Go Modules的开关配置被称为GO111MODULE。1.13版本之后，Go Modules被设置为Go默认的包管理工具。对于它的值，我觉得无脑on就行了。当然它除了off之外，还有个auto的选项，我个人觉得没什么太大的意义。

### 前置概念：package 和 module

- package即go的代码包，go代码的第一行就声明了改代码归属的代码包。代码包可以分为：
  - Go标准包
  - 第三方包，比如来自github.com的包
  - 匿名包，import _ ***，主要执行被引入包的init函数
  - 内部包，同一项目中的其他代码包
- module(模块)是package的集合

- 在Go Modules中，可以认为每一个项目都是一个module，它的根目录下都有一个go.mod文件。它用来定义了本module的的名称及其所依赖的其他module，它的具体含义后面会分析。

### go mod命令

- 先简单列出几个关键的命令，它们的具体使用和逻辑会在后文中体现出来。

| 命令     | 作用                                                         |
| -------- | ------------------------------------------------------------ |
| init     | 将当前目录初始化成一个新module                               |
| download | 下载所有依赖包                                               |
| graph    | 查看依赖结构                                                 |
| tidy     | 添加丢失的模块（项目中import了，gomod文件中没有的），移除无用的模块（相反） |
| vendor   | 将所有依赖包保存到当前项目下的vendor目录下。                 |
| verify   | 检查当前模块的依赖是否已经存储在本地下载的源代码缓存中，以及检查下载后是否有修改。 |
| why      | 查看为什么需要依赖某一个模块                                 |

### Go Modules的下载

- 开启了Go Module后，执行go get不管是步骤还是结果都和GOPATH模式下不太一样。本节主要讨论go get的过程，结果在后面讨论。

#### 下载源的变化

  - Go Module中多了一个环境变量 :**GOPROXY**。用于设置Go模块代理（下载代理）。
  - 配置好GOPROXY后，go get命令就会先从GOPROXY中下载代码了，这能解决很多问题，比如：
    - 有些代码仓库被墙了或者访问特别慢，GOPROXY设置为国内可访问的地址就可以避免问题。一般使用七牛提供的https://goproxy.cn。
    - 有些代码仓库中的代码可能被删除，而代理服务器会永久存储，解决丢失风险。
    - 有的公司还能自己搭建代理服务器，速度更快。
  - GOPROXY可以配置多个代理服务器，用,隔开。下载时会按顺序获取。
  - 另外GOPROXY可以配置一个特殊值：direct，代表回源。既直接去go get后面的真实路径去获取。如将GOPROXY设置为https://goproxy.cn,direct，go get github.com/gin-gonic/gin时，会先去https://goproxy.cn下载代码，如果失败，就回源去github.com/gin-gonic/gin下载。
  - 另外还有两个相关环境配置：**GONOPROXY**和**GONOSUMDB**。
    - GONOPROXY是针对GOPROXY的。项目开发中会有一些私有仓库从代理服务器上是获取不到的，比如公司内部的gitlab上的代码，这就需要配置GONOPROXY。比如将GONOPROXY配置成公司的gitlab地址，当拉取公司的gitlab上的代码的时就不会走GOPROXY了。
    - GONOPROXY上拉回来的代码自然不能走检验步骤了，配置GONOSUMDB后就会绕开检验。检验的逻辑后面有详细的分析。
    - 很明显GONOPROXY和GONOSUMDB的值一般情况下会是一样的，有环境配置**GOPRIVATE**的值可做为它们的默认值，所以只要配置好GOPRIVATE就行了。

#### 不同版本下载

- 还可以通过指定版本来下载模块，go get package@verison。比如下载gin的v1.7.6版本。那就是执行 go get github.com/gin-gonic/gin@v1.7.6。
- 开发中经常会遇到一个模块依赖了很多其他模块，不同的模块又依赖的同一个模块的不同版本。
  - 举个例子来说明Go Modules时如何处理的，以下用字母表示一个模块，->表示依赖。
  - 主体的依赖关系：A->B,C  | B->D  | C->D,F | D -> E。而每一个模块根据版本号又有不同的代码版本。
  -  Go Modules是这样选择版本的：
    - 从A出发总共有3条依赖链路：
      - A->B->D->E
      - A->C->D->E
      - A->C->F
    - 这3条依赖链路里面，可能产生版本分歧的就是D。因为B，C都依赖了D，就可能会出现依赖不同版本的D。
    - 这个时候，Go Modules会选择高版本的D。因为
      - 如果是小版本的差别，比如v1.1.0和v1.2.0。按照规范，高版本应该兼容低版本。
      - 如果是大版本的差别，比如v1.0.0和v2.0.0。那它们的模块导入路径应该不同！这里插个眼，在描述gomod文件时会再次提到这里。



### go.mod文件

- 在这个文件里面，定义了模块的名称及其所依赖的包，而每一个依赖包都需要指定导入路劲和语义化版本来准确的描述。

```
module gitlab.xxx.com/hello/gomodules

go 1.17

require (
	github.com/dgrijalva/jwt-go v3.2.0+incompatible
	github.com/gogf/gf v1.16.6
	github.com/go-sql-driver/mysql v1.6.0 // indirect
	gitlab.com/wang/utils v1.1.0
	gopkg.in/yaml.v3 v3.0.0-20210107192922-496545a6307b // indirect
)

replace ( 
	gitlab.com/wang/utils => /home/wang/utils 
)

exclude ( 
	github.com/gogf/guuid v1.0.0
)
```

- 这是随便凑的一个gomod文件，目的是多覆盖关键字便于分析它们的意义
- 先列出来出现的关键字：
  - module
  - go
  - require
  - replace
  - exclude
  - vx.x.x
  - +incompatible
  - indirect

#### module

- 用于声明本module的名字。也是本module的模块路径。

#### go

- 声明本模块的go版本，仅仅是声明而已，没有实际用处。

gomod命令章节在这里卡个位：在新建项目时，会使用go mod init xxx来初始化Go Modules。这条命令会创建一个go mod文件，并且会使用init后面的xxx作为module名，将开发环境的go版本作为gomod文件中的go版本。

#### require

- 设定特定的依赖模块，格式为 [模块路径] [版本]
- 版本的细节在vx.x.x中描述。

gomod命令章节再卡个位：在开发项目时，需要引入外部模块代码时，通常会先在代码中import外部模块的路径，然后使用go mod tidy命令，这时会自动下载代码并且更新gomod文件，但是这种不能指定版本。需要指定版本的情况下可以在gomod文件中的require部分先写好模块路径和版本，再执行go mod tidy命令。

#### replace

- 用于将require中的指定的引入路径替换，格式为 [module] => [newmodule]
- 这样下载包时,就会去newmodule下载.
- newmodule可以是网络地址，也可以是本地路径。replace主要用来解决拉取不到的问题，比如网络不通或者下载权限。

#### exclude

- 用于排除一个模块。

注意：replace和exclude中在当前模块生效，不会影响到当前模块所依赖的其他模块。

#### vx.x.x

- 版本号
- 如果被依赖的模块有语义化版本格式的tag，那就会直接展示tag的值。如 github.com/gogf/gf v1.16.6。v1.16.6就是版本号。
- 当没有语义化版本格式的tag时，Go命令会选择master分支上最新的commit计算出一个值，如gopkg.in/yaml.v3 v3.0.0-20210107192922-496545a6307b 。

#### +incompatible

- 如果一个模块的大版本时v0或者v1，那它的模块路径没有特殊要求，如github.com/gogf/gf v1.16.6。
- 但是大版本号大于v1时，那么大版本就必须要出现在模块路径的尾部，如github.com/appleboy/gin-jwt/v2 v2.6.3。如果没有，当我们go mod tidy的时候，就会在版本号后面自动加上+incompatible，如github.com/dgrijalva/jwt-go v3.2.0+incompatible。TP到Go Modules的下载那一节中的不同版本下载部分。

#### indirect

- 表示不是直接依赖的模块。非直接依赖的原因有两个：

  - 直接依赖没有开启Go Modules的时候，比如A依赖B，B又依赖了B1，B2。但是B没有go.mod文件，那么B1，B2会记录到A的go.mod中，

    并加上// indirect。

  - 直接依赖go.mod文件中有缺失，比如A依赖B，B又依赖了B1，B2。B虽然有go.mod文件，但只有B1被记录在B的go.mod文件中，这时候B2就会记录在A的go.mod中，并加上// indirect。

- 我自己的开发环境时go的1.17版本，我发现go mod tidy的时候，会把indirect的依赖分开写在另一个require部分中，和直接依赖做区分。

### go.sum文件

- 这个文件是我早期使用中觉得最没存在感的文件，但是仔细研究它，它的作用很大的。
- 在讨论它之前，先来解释下，开启Go Modules之后，go get到的代码去了哪里。以执行go get github.com/gin-gonic/gin@v1.7.6为例：

  - 执行go get命令后，$GOPATH/pkg/mod/cache/download/github.com/gin-gonic/gin/@v下会出现几个文件：
    - v1.7.6.zip：代码的压缩包
    - v1.7.6.ziphash：代码的hash值
    - v1.7.6.mod：模块的gomod文件
    - v1.7.6.info：模块的基本信息
  - 另外，会创建$GOPATH/pkg/mod/github.com/gin-gonic/gin@v1.7.6文件夹，保存着代码。
  - 从上诉过程中描述的代码存储目录，明显是可以保存多个版本的。
- 仔细想想，目前的Go Modules会有哪些问题？
  - 当刚刚用git拉回来一个项目时，拉回来了项目代码和go.mod文件。然后执行go mod tidy下载项目依赖的其他模块。但是下回来的源代码被篡改了，也就是和项目开发者当时依赖的代码不一样了怎么办？这有可能导致项目出问题。
  - 或者是代码已经存在于$GOPATH/pkg/mod下了，但是也有被改的可能性。怎么处理？
    - 虽然整个$GOPATH/pkg/mod都被设置成了只读模式，但是还是没办法完全避免被篡改。

- go.sum就是来解决问题的。

- 这是一个被依赖模块对应的go.sum文件：

  ```
  github.com/BurntSushi/toml v0.3.1 h1:WXkYYl6Yr3qBf1K79EBnL4mak0OimBfB0XUf9Vl28OQ=
  github.com/BurntSushi/toml v0.3.1/go.mod h1:xHWCNGjB5oqiDr8zfno3MHue2Ht5sIBksp03qcyfWMU=
  ```

  - 一个依赖模块一般有两行：
    - [][]第一行：[module] [version] [hash algorithm ] :[hash]。只需注意h1指使用的hash算法，SHA-256。
    - 第二行：[module] [version/go.mod] [hash algorithm ] :[hash]。第一行是对代码做hash，第二行是对gomod文件做hash。当然这个模块没有gomod文件时，就没有第二行。

- go.sum中的内容是如何生成的？

  - 当在项目根目录下使用go get命令下载依赖包的时候，除了上面说的结果，还会更新go.mod和go.sum文件。
  - 从go get下载到更新go.sum文件中还有一个很关键的步骤，这会涉及到一个环境配置**GOSUMDB**。
    - GOSUMDB的值是一个web服务器，默认为sum.golang.org。该服务可以查询依赖包指定版本的hash值。也就是说比如go get gin的v1.7.6的代码时，会计算出代码的hash值，然后和sum.golang.org上查询的结果做对比，不一样时，就认为go get的代码是被篡改过的。
    - 当然GOSUMDB服务器，可以自己搭建。也可以配置成off，当然这样有风险。
    - 所以，当go.sum中更新了内容时，就可以认为是被检验过的。它的hash值是可信的。
  - 通过这一通操作，go.sum中的hash值就是用来做对比的基准。

- 这样Go Modules的问题就解决了。在项目目录下go get操作的时候有两种情况：

  - 如果代码不在本地，先下载，再通过GOSUMDB检验是否被篡改，通过检验则缓存本地
    - go.sum不存在模块信息时，则录入。
    - 存在时，做比较，不一致就报错。(已经存在的go.sum是检验的基准）
  - 如果模块代码已经在本地：
    - go.sum不存在模块信息，则录入。（下载过程中已经被GOSUMDB检验了，而本地缓存时只读的，所以被认为是可靠的，可直接写入go.sum）
    - 存在时，做比较，不一致就报错。

###   internal包

- 自己的模块是会被其他模块所依赖的，当有一些代码不希望其他模块使用时，就可以在自己的模块内创建internal代码包，包内的代码其他模块就引用不到。

  

  
