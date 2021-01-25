---
title: "Go模块化编程"
date: 2019-02-26
categories: ['note', 'tech']
draft: false
---

Go语言从去年八月份发布的1.11版本增加了模块化编程的支持以及新的基于模块的依赖管理工具。Go语言的模块（module）是文件树中的包（package）的集合，其中模块根目录包含的`go.mod`文件定义了模块的导入路径（import path），Go语言版本以及模块的其他依赖性要求。每个模块依赖性要求被列为单独的一个模块路径并指定相应的模块版本，只有满足了所有依赖性要求的模块才能被成功构建。

使用Go模块化编程之后，就不需要将Go语言的代码放入到`$GOPATH/src`中，也就是过去的`GOPATH`模式，实际上，我们可以在`$GOPATH/src`外的任何目录下使用`go mod init`创建Go项目并初始化Go模块。

> Note: 为了兼容性，在Go 1.11与1.12中，Go命令仍然在旧的`GOPATH`模式下运行，从Go 1.13开始，模块模式(`GO111MODULE=on`)将成为默认模式。

## GOPATH的前世今生

而Go语言支持模块化编程之前，一般我们的Go项目使用需要使用`GOPATH`模式，也就是说需要将Go语言的代码放入到`$GOPATH/src`中。典型的`GOPATH`目录结构包含必须包含三个子文件夹，如下所示：

```
GOPATH
├── bin              // binaries
├── pkg              // cache
|── src              // go source code
    ├── github.com
    ├── rsc.io
    ...
```

使用`go get`命令获取依赖也会自动下载到`$GOPATH/src`中：

```
go get rsc.io/quote # will be doloaded to $GOPATH/src/rsc.io/quote
```

但是`GOPATH`模式存在问题，当我们`go get`获取某个依赖时没有指定版本，那么默认下载的依赖代码都会是最新版本，而且如果当项目A和项目B分别依赖项目C的两个不兼容版本时，`GOPATH`路径下只有一个版本的C将无法同时满足项目A和项目B的依赖需求。这是令人棘手的缺陷，实际上因而Go 1.13开始，官方就不再推荐使用`GOPATH`模式了。

另外，随着Go语言的普及，依赖包也变得越来越丰富，依赖管理的问题逐渐成为开发人员的焦点。`GOPATH`模式衍生出了很多版本管理工具，但是他们的基本思路都是基于**每个项目单独维护一份对应版本依赖的拷贝**：

1. 从最早Go官方在Go 1.5版本提出的`vendor`特性：为每个Go项目创建一个`vendor/`目录来存放项目所需版本依赖的拷贝
2. 社区则基于Go官方的`vendor`特性，开发出了各种版本管理工具。比较流行的如[govendor](https://github.com/kardianos/govendor)，以及之前曾被官方认定的[godep](https://github.com/golang/dep)等

Go依赖管理工具虽然丰富了起来，但是不同版本工具之间存在不兼容的问题，而且各种工具还都有学习成本。这时候开发者迫切需要一个统一的版本管理工具来解决这种问题。于是Go社区提出了[vgo](https://github.com/golang/go/wiki/vgo)方案，随着`vgo`的逐渐发展壮大，Go 1.11发布了基于改方案的Go module功能，并集成到了Go语言的官方工具中。


## Go模块化支持的相关环境变量

使用Go语言内置的模块化功能，开发者需要关心的6个环境变量，可以使用`go env`命令列出：

```
GO111MODULE="auto"
GOPROXY="https://goproxy.io,direct"
GONOPROXY=""
GOSUMDB="sum.golang.org"
GONOSUMDB=""
GOPRIVATE=""
```

1. GO111MODULE

该环境变量控制是否开启使用Go模块功能，可选的值以及说明如下表所示：

｜ 值 | 说明 |
| --- | --- |
| auto | `$GOPATH/src`之中的项目继续使用`GOPATH`模式；`$GOPATH/src`之外的Go项目自动使用Go模块化模式 ｜
| on | 启用Go模块化模式，推荐使用 ｜
| off | 使用`GOPATH`模式，禁用Go模块化模式，不推荐设置 ｜

开启Go模块化模式只需要设置`GO111MODULE`环境变量为`on`：

```
$ go env -w GO111MODULE=on
```

2. GOPROXY

该环境变量用于设置Go模块依赖下载的代理，这样Go在拉取模块依赖时直接通过代理站点来快速拉取。默认值是`https://proxy.golang.org,direct/`。由于国内无法访问，所以开启Go模块化模式需要设置代理地址。常用的国内镜像代理地址如下：

| 地址 | 简介 |
| --- | --- |
| `https://goproxy.io` | 最早的Go模块镜像代理网站 |
| `https://mirrors.aliyun.com/goproxy/` | 阿里镜像代理网站 |
| `https://goproxy.cn` | 七牛云赞助支持的代理网站 ｜

> Note: `GOPROXY`的值是一个以英文逗号`,`分割的Go模块代理列表，允许设置多个模块代理；如果想关闭代理，可以设置为`off`

默认值的逗号后面的`direct`用来告诉Go语言将直接从依赖的源地址进行下载（如GitHub），例如当值列表中的Go镜像代理返回`404`或`410`时，Go自动尝试列表中的下一个，如果下个值是`direct`则到源地址去下载依赖的模块。

设置Go模块化镜像代理：

```
$ go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/,direct
```

3. GOSUMDB

该环境变量的全称是GO checkSUM DataBase，正如它的字面意思一样，它用于在拉取模块依赖时保证拉取到的模块版本数据未经过篡改，若发现不一致，则立即中止拉取并报错。
`GOSUMDB`环境变量的默认值为：`sum.golang.org`，在国内也是无法访问的，好在设置好可以访问的`GOPROXY`之后，`GOSUMDB`也可以访问。如果将它的值若设为`off`，就禁止Go在后续拉取模块依赖时进行校验，不推荐这样做。

4. GONOPROXY & GONOSUMDB & GOPRIVATE

这三个环境变量主要在Go项目依赖于私有模块时使用，比如有些依赖存在于私有的git仓库，直接使用`GOPROXY`设置的镜像代理或`GOSUMDB`设置的检验站点会无法访问对应的私有仓库。这时候就需要设置这些依赖模块不经过镜像代理站点直接拉取校验。实际上，对于私有的依赖模块，最佳实践是直接设置`GOPRIVATE`环境变量，它的值将作为`GONOPROXY`和`GONOSUMDB`的默认值。

它们的值都是一个以英文逗号`,`分割的模块路径前缀，也就是可以设置多个，例如：

```
$ go env -w GOPRIVATE="*.my.com,github.com/eabc/def"
```

这项设置表示前缀为`*.my.com`和`github.com/eabc/def`的模块依赖都会被认为是私有模块，并不经过镜像代理站点直接拉取校验。

## Go模块化操作命令

1. `go mod init`创建一个新模块，初始化描述它的`go.mod`文件
2. `go build` & `go test`和其他程序包构建命令根据需要向`go.mod`添加新的依赖项
3. `go list -m all`打印当前模块的所有依赖关系
4. `go get`更改所需依赖的版本（或添加新的依赖）
5. `go mod tidy`删除未使用的依赖项
6. `go mod vendor`从模块依赖的正确版本拷贝到项目的vendor目录

1. 初始化模块

我们可以在`$GOPATH/src`外的任何目录下使用`go mod init`创建Go项目并初始化Go模块：

```
$ go mod init example.com/greetings
go: creating new go.mod: module example.com/greetings
```

`example.com/greetings`不仅是模块的标识，作为模块的导入路径，当其他Go项目引用这个模块下的某个包时都会以该导入路径作为共同的前缀，并加上该包相对于模块根目录的相对路径。

2. `go.mod`文件

上面的命令成功执行之后就会在当前目录下面生成一个`go.mod`文件：

```
module example.com/greetings

go 1.12
```

实际上，Go模块化模式下，使用`go get`命令获取模块依赖之后，相关依赖信息可以自动记录到`go.mod`文件中：

```
go get -u rsc.io/quote
```

`go get`命令默认下载最新的依赖版本，当然也可以通过`@`版本管理的tag，例如：

```
go get -u rsc.io/quote@v1.5.2
```

`go get`命令拉取模块依赖之后会将结果缓存到`$GOPATH/pkg/mod`和`$GOPATH/pkg/sumdb`目录，而在`$GOPATH/pkg/mod`目录中模块依赖会以`github.com/foo/bar`的格式进行存放。

拉取模块依赖之后，`go.mod`文件会变成：

```
module example.com/greetings

go 1.12

require rsc.io/quote v1.5.2 // indirect
```

> Note: 其中，`indirect`注释表示该模块为间接依赖，也就是在当前应用程序中的`import`语句中，并没有发现这个模块的明确引用。直接使用`go get`命令拉取模块依赖，而不是使用`go build`自动根据`go.mod`拉取模块依赖。

`go.mod`文件中可以使用到的语法关键词以及含义：

- `module`：定义当前项目的模块路径
- `go`：标识当前模块的Go语言版本
- `require`：说明模块依赖的版本
- `exclude`：用于从使用中排除一个特定的模块版本，如果某个版本的模块依赖有严重bug，则显式的排除某个版本，`exclude github.com/google/uuid v1.1.0`表示不使用`v1.1.0`版本
- `replace`：替换`require`中声明的模块依赖，使用另外的模块依赖及其版本

`replace`关键词的使用场景：

**使用场景一：替换require的包**

当前项目的`go.mod`文件如下

```
module example.com/greetings

go 1.12

require (
    github.com/google/uuid v1.1.1
    rsc.io/quote v1.5.2 // indirect
)

exclude github.com/google/uuid v1.1.0
```

下执行命令`go list -m all`列出当前模块的所有依赖关系：

```
example.com/greetings
github.com/google/uuid v1.1.1
rsc.io/quote v1.5.2
```

在`go.mod`文件下增加这样一句配置：

```
replace github.com/google/uuid v1.1.1 => github.com/google/uuid v1.1.0
```

再次执行命令`go list -m all`列出当前模块的所有依赖关系：

```
example.com/greetings
github.com/google/uuid v1.1.1 => github.com/google/uuid v1.1.0
rsc.io/quote v1.5.2
```

发现最终生效的模块依赖从`github.com/google/uuid v1.1.1`变成了`github.com/google/uuid v1.1.0`。一般来说，这种场景使用的不多，和直接修改`require`作用相同，其生效有前提条件：

- 当前引用的模块依赖有效
- `replace`命令左侧的包名和版本，必须是`require`中包含的对应包名和版本

**场景二：替换无法下载的包**

由于国内网络限制问题，有些模块依赖包无法下载，比如`golang.org`下的所有依赖包，好在这些依赖包在GitHub都有对应的镜像，此时就可以使用GitHub上的镜像包来替换`golang.org`导入路径下的模块依赖包。

比如，如果项目中使用了`golang.org/x/net`包：

```
module example.com/greetings

go 1.12

require (
    github.com/google/uuid v1.1.1
    golang.org/x/net v0.3.2
    rsc.io/quote v1.5.2 // indirect
)

replace golang.org/x/net v0.3.2 => github.com/golang/net v0.3.2
```

这样项目编译时就会从GitHub下载`golang.org/x/net`包，而在源代码中`import`路径`golang.org/x/net/xxx`则不需要改变。

**场景三：调试依赖包**

场景三：调试依赖包


调试依赖包时可以使用`replace`来修改依赖，如下所示：

```
replace (
    github.com/google/uuid v1.1.1 => ../uuid
)
```

使用本地的`uuid`来替换依赖包，此时，我们可以任意地修改`…/uuid`目录的内容来进行调试。除了使用相对路径，还可以使用绝对路径，甚至还可以使用自已的fork仓库。

**场景四：禁止被依赖**

最后一种使用`replace`的场景是不希望自己的模块被直接引用，比如kubernetes项目的的`go.mod`文件中，`require`部分有大量的`v0.0.0`依赖，比如：

```
module k8s.io/kubernetes

require (
  ...
  k8s.io/api v0.0.0
  k8s.io/apiextensions-apiserver v0.0.0
  k8s.io/apimachinery v0.0.0
  ...
)
```

由于上面的依赖都不存在`v0.0.0`版本，所以其他项目直接依赖`k8s.io/kubernetes`时会因无法找到版本而无法使用。

因为`Kubernetes`不希望自己作为模块依赖被直接使用，其他项目可以使用kubernetes的其他子组件。kubernetes对外隐藏了依赖版本号，其真实的依赖通过`replace`指定：

```
replace (
  k8s.io/api => ./staging/src/k8s.io/api
  k8s.io/apiextensions-apiserver => ./staging/src/k8s.io/apiextensions-apiserver
  k8s.io/apimachinery => ./staging/src/k8s.io/apimachinery
    ...
)
```

3. `go.sum`文件

在我们构建执行`go build`或者`go test`等命令后还会发现在项目的根目录下面生成个`go.sum`文件。它的主要用途是检验下载的模块依赖包。实际上，模块依赖包在下载过程中有可能被恶意篡改，缓存在本地的依赖包也有被篡改的可能，单单一个`go.mod`文件并不能保证一致性构建，在`go.mod`的基础上同时也引入了`go.sum`文件，用于记录每个依赖包的哈希值（SHA-256 算法），在构建Go项目时，如果本地的依赖包哈希校验值与`go.sum`文件中记录得不一致，则会拒绝构建。

一个典型的`go.mod`文件如下所示：

```
github.com/google/uuid v1.0.0 h1:b4Gk+7WdP/d3HZH8EJsZpvV7EtDOgaZLtnaNGIu1adA=
github.com/google/uuid v1.0.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
```

正常情况下，每个依赖包版本会包含两条记录，第一条记录为该依赖包版本整体（所有文件）的哈希值，第二条记录仅表示该依赖包版本中`go.mod`文件的哈希值，如果该依赖包版本没有`go.mod`文件，则只有第一条记录。


go.sum是怎么生成的？

在Go项目的根目录中执行`go get`命令，它会同步更新`go.mod`和`go.sum`文件，`go.mod`中记录的是依赖名及其版本，如：

```
github.com/google/uuid v1.1.1
```

`go.sum`文件中则会记录依赖包的哈希值（同时还有依赖包中`go.mod`的哈希值）。在更新`go.sum`之前，为了确保下载的依赖包是真实可靠的，go命令在下载完依赖包后还会查询`GOSUMDB`环境变量所指示的服务器，以得到一个权威的依赖包版本哈希值。如果go命令计算出的依赖包版本哈希值与`GOSUMDB`服务器给出的哈希值不一致，go命令将拒绝向下执行，也不会更新`go.sum`文件。

## Go模块的使用与更新

我们首先使用`go mod init`命令初始化Go项目的模块并设置模块的导入路径，然后在我们写代码的时候先写好导入的包等，`go build`或者`go mod tidy`等命令会自动下载相关导入包并更新`go.mod`与`go.sum`文件，这时候使用版本管理工具比如Git提交并管理项目源代码与更新后的`go.mod`与`go.sum`文件，确保每次的构建都可以得到一致的依赖关系。

1. 小版本更新

使用`go get -u`命令会更新某个模块依赖最新的小版本，例如，它会将`1.0.0`更新为`1.0.1`或者`1.1.0`这样类似的版本；`go get -u=patch`命令则是获取最新的patch更新，例如，它会将`1.0.0`更新到`1.0.1`而不是`1.1.0`版本：

假如我们的Go项目使用的是依赖包的`1.0.0`版本，并且该模块依赖刚刚更新了`1.0.1`版本，以下任何命令都会将我们的更新到模块依赖`1.0.1`版：

```
go get -u
go get -u=patch
go get github.com/example/testmod@v1.0.1
```

2. 大版本更新

一般来说，大版本完全不同于小版本，大版本可能会破坏向后兼容性。因此，从Go模块的角度来看，大版本是一个完全不同的包。一个库的两个不兼容的版本，实质上是两个不同的库。遇到大版本更新的时候，还是可以使用tag来更新，从`1.0.0`更新为`2.0.0`，但是一定要注意向后兼容性的问题。
