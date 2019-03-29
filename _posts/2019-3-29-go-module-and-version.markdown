---
layout: default
title:  "go module和版本管理"
date:   2019-03-29 12:23:38
---

本文主要介绍 go 1.12中引入的 module 管理。首先是介绍一种称为语义化的版本描述。然后介绍使用 go mod 需要使用的工作流。包括：
1. 初始化一个 go mod。
2. 如何引入第三方的 module。
3. 如何更新依赖 module。
4. 如何升级依赖 module。
5. 清理依赖 module。

本文大部分内容为翻译 Using Go Modules https://blog.golang.org/using-go-modules 。

## 语义化版本 

go module 的版本管理基于一种叫作语言化版本的定义，其详细描述可参考 https://semver.org 。这里只作简要介绍。
一个语义化的版本定义为 `x.y.z` ，对应于 `主版本号.次版本号.修订号`，其编写的规则如下：
- 如果出现不兼容的功能升级，主版本号增加。
- 相同主版本号的接口是兼容的，如果有接口增加，增加次版本号。
- 仅修复问题，如Bug fix，则仅增加修订号。

## Go module 工作流

### 初始化 

启用 module 管理后，`GOPATH` 不再用于解析 import 路径。但也没有完全弃用，而是将引入的其它mod放置于`GOPATH/pkg/mod`下，编译的结果置于`GOPATH/bin`。新的项目不能在 `GOPATH` 中。参考以下代码新建一个go module.

```bash
➜  ~ cd ~/
➜  ~ mkdir helloworld
➜  ~ mkdir ~/.gopkgs
➜  ~ export GOPATH=$HOME/.gopkgs
➜  ~ cd helloworld
➜  go mod init github.com/icattlecoder/helloworld
go: creating new go.mod: module github.com/icattlecoder/helloworld
```
这样我们就创建了一个路径为 `github.com/icattlecoder/helloworld` 的 module。然后我们添加一个`lib/foo.go` 文件，内容如下：

```go
package lib

func Hello() string {
	return "hello, world"
}
```

在项目根路径添加 main.go文件，内容如下：

```go
package main

import (
	"github.com/icattlecoder/helloworld/lib"
)

func main() {
	print(lib.Hello())
}
```

整个项目的路径如下：

```
➜  tree
.
├── go.mod
├── lib
│   └── foo.go
└── main.go
```

执行 `go install ./...` 后，`GOPATH/bin`下生成了 `helloworld` 可执行文件。
我们发现，**虽然我们在main.go文件中引入了`github.com/icattlecoder/helloworld/lib`包，但是我们的项目其实并不在这个路径下面。也就是项目路径跟 import 路径解耦了。**

如果这个项目不使用go module, 那么就要将其置于路径 `$GOPATH/src/github.com/icattlecoder/helloword` 下。


### 引入依赖module

main.go的内容改为以下:
```go
package hello

import "rsc.io/quote"

func main() {
	println(quote.Hello())
}
```
执行 `go install ./...`，go 自动下载依赖包 `rsc.io/quote`。

```
➜  go install  ./...
go: finding rsc.io/quote v1.5.2
go: downloading rsc.io/quote v1.5.2
go: extracting rsc.io/quote v1.5.2
go: finding rsc.io/sampler v1.3.0
go: golang.org/x/text@v0.0.0-20170915032832-14c0d48ead0c: unrecognized import path "golang.org/x/text" (https fetch: Get https://golang.org/x/text?go-get=1: dial tcp 216.239.37.1:443: i/o timeout)
go: error loading module requirements
```
没有翻墙的情况下会出现上面的错误提示，go mod 有一种非常简单的解决方案，作如下设置
```
export GOPROXY=https://goproxy.io
```
继续执行 `go install ./...`，即可下载所有依赖。

这里我们可以看到，**`go install `,`go build`,`go test`等工具都会自动下载依赖，即使在项目中仅仅import而没有使用也可以。**

### 次版本更新

这里的更新是指`次版本号`或者`修订号`的升级，即更新后程序依旧兼容，比如一个依赖包修复了一个BUG或者优化了性能。
我们先看下项目所有的依赖包：
```
➜  go list -m all
github.com/icattlecoder/helloworld
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
```
其中 `golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c` 采用的是伪tag，形如 `v0.0.0-yyyymmddhhmmss-abcdefabcdef` ，用来描述没有打 tag的代码。`yyyymmddhhmmss`是commit的提交时间，`abcdefabcdef`是 commit log。现在我们用`go get`下载最新版本的 `golang.org/x/text`
```shell
➜  go get golang.org/x/text
go: finding golang.org/x/text v0.3.0
go: downloading golang.org/x/text v0.3.0
go: extracting golang.org/x/text v0.3.0
```
编译 `go install ./...` ，没有问题。 看下 go.mod 文件:
```
module github.com/icattlecoder/helloworld

go 1.12

require (
	golang.org/x/text v0.3.0 // indirect
	rsc.io/quote v1.5.2
)
```
多了一行`golang.org/x/text v0.3.0 // indirect` ，`indirect ` 表示这个包非当前项目直接引用。

正常情况下， `go get`应该不影响兼容性，但是万一出现了兼容性问题，可以通过指定版本号来解决。

**使用 `go list -m -versions` 查看版本列表。**

```
➜  go list -m -versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99
```

**使用 `go get pkgPath@version` 下载指定版本包。**

```
➜  go get rsc.io/sampler@v1.3.1
```

### 主版本更新

主版本更新是指主版本号的增加，即程序出现了不兼容的升级。通常，对于同一个主版本号，程序不可能出现多种次版本或修订号。而对于不同的主版本号是可以并存的。go 中建议不同主版本的导入路径应该不同，又称语义化导入，关于这种方式的详细描述可以参考：[semantic import versioning](https://research.swtch.com/vgo-import)。

修改`main.go`为以下内容：
```go
package hello

import (
	"rsc.io/quote"
	quoteV3 "rsc.io/quote/v3"
)

func main() {
	println(quote.Hello())
	println(quoteV3.Concurrency())
}
```
有几点需要注意一下：

1. `rsc.io/quote/v3` 的路径不同于 `rsc.io/quote`，当是包名其实相同，这就需要给`v3`另起一个别名 `quoteV3`
2. `v3`中多出了一个 `v1` 版本中没有的函数 `Concurrency`。

执行`go install  ./...`:
```
➜  go install ./...
go: downloading rsc.io/quote/v3 v3.1.0
go: extracting rsc.io/quote/v3 v3.1.0
```
### 清理依赖
现在我们去掉	`rsc.io/quote`的引入，`main.go`函数改为：
```
package hello

import 	"rsc.io/quote/v3"

func main() {
	println(quote.Concurrency())
}
```

查看 `go.mod` 内容：
```
module github.com/icattlecoder/helloworld

go 1.12

require (
	golang.org/x/text v0.3.0 // indirect
	rsc.io/quote v1.5.2
	rsc.io/quote/v3 v3.1.0
)
```
我们发现 `rsc.io/quote v1.5.2` 不再引入，可以通过执行`go mod tidy`清理后，再查看 `go mod`内容
```
module github.com/icattlecoder/helloworld

go 1.12

require (
	golang.org/x/text v0.3.0 // indirect
	rsc.io/quote/v3 v3.1.0
)
```
发现 `rsc.io/quote v1.5.2`已经被移除。
当然，也可以通过`go list -m all` 查看项目所有的依赖。

## 非github路径是如何引入的

上面的示例中，`rsc.io/quote`包是如何引入的呢？
首先，我们直接访问下`rsc.io/quote`:
```sh
root@ubuntu:~# curl -i rsc.io/quote
HTTP/1.1 302 Found
Location: https://rsc.io/quote
```
发现他是跳转到 `https://rsc.io/quote` ，继续看下他的内容：
```html
root@ubuntu:~# curl https://rsc.io/quote
<!DOCTYPE html>
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
<meta name="go-import" content="rsc.io/quote git https://github.com/rsc/quote">
<meta http-equiv="refresh" content="0; url=https://godoc.org/rsc.io/quote">
</head>
<body>
Redirecting to docs at <a href="https://godoc.org/rsc.io/quote">godoc.org/rsc.io/quote</a>...
</body>
</html>
```
看到下面这一行：
```html
<meta name="go-import" content="rsc.io/quote git https://github.com/rsc/quote">
```
不难理解，他指示 go 使用 git 从 https://github.com/rsc/quote 下载代码到 rsc.io/quote 目录下。