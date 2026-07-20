---
title: Go 语言教程系列のHello World
date: 2023-03-19 15:02:00
cover: https://cdn.jsdelivr.net/gh/youngjuning/images@main/1679280067130.png
description: 学习编程语言没有比让自己满手沾满代码的血🩸更有效的方法了。让我们一起开始第一个 Go 程序吧。
categories:
  - [Go]
  - [紫升翻译计划]
tags:
  - Go
  - Go 教程
  - Go 教程
---

这是《Golang 教程系列》的第二篇。如果还没有，你可能需要先看一下我们的上一篇教程 [Golang 介绍和环境安装](https://juejin.cn/post/6942492190291525662) 以了解一下 Golang 是什么以及如何下载 Golang。

这篇教程是基于 Go 1.16 以上的版本。

学习编程语言没有比让自己满手沾满代码的血🩸更有效的方法了。让我们一起开始第一个 Go 程序吧。（First Blood）

## 设置开发环境

让我们先创建一个编写 hello world 程序的文件夹。请打开命令行然后运行下面的命令：

```sh
mkdir ~/Desktop/learngo/
```

上面的命令将会在用户桌面创建一个名为 `learngo` 的文件夹（译者开发环境为 Macbook Pro）。你可以在任意位置创建目录编写代码。

## Hello World

使用你喜爱的文本编辑器在 `learngo` 文件夹中创建一个名为 `main.go` 的文件并写入以下内容：

```go
package main

import "fmt"

func main() {
  fmt.Println("Hello World")
}
```

在 Go 中将包含 `main` 函数的文件命名为 `main.go` 是约定俗称的，但是其他名称也是可以使用。

## 运行一个程序

有几种不同的方式来运行 Go 程序。让我们一一看一下。

### 1. `go install`

第一个运行 Go 程序的方法是使用 `go install` 命令。让我们来使用 `cd` 命令进入刚创建的 `learngo` 目录

```sh
cd ~/Desktop/learngo/
```

接着运行下面的命令。

```sh
go install
```

上面的命令将编译当前程序并将其安装（拷贝）二进制可执行文件到 `~/go/bin` 目录。二进制可执行文件的名字是包含 `main.go` 文件的文件夹名。在我们示例中，它将被命名为 `learngo`。


当你尝试安装程序时，你可能会遇到以下错误。

```sh
go: cannot find main module; see 'go help modules'
```

上面的错误实际上意味着，`go install` 无法找到 main 模块，这是因为我们没有初始化 go modules，我们使用以下命令初始化模块：

```sh
go mod init github.com/youngjuning/learngo
```

上面的命令会在 `learngo` 目录下创建一个 `go.mod` 文件，该文件是程序模块定义的地方，作用类似于 Node 的 `package.json` 文件。然后我再执行 `go install` 便可以成功。

你可以在命令行输入 `ls -al ~/go/bin/learngo`，然后你会发现 `go install` 实际上是把二进制可执行文件放在了 `~/go/bin` 中。

现在让我们运行编译后的二进制可执行文件。

```sh
~/go/bin/learngo
```

上面的命令将运行 `learngo` 并打印出以下内容：

```sh
Hello World
```

恭喜你！你已经成功地运行了你的第一个 Go 程序。

如果你不想每次都输入完整的 `~/go/bin/learngo` 路径来运行程序，你可以添加 `~/go/bin/` 到你的 `PATH` 中。

```sh
export GOPATH=~/.go
export PATH=${PATH}:$GOPATH/bin
```

现在你可以在命令行中只输入 `learngo` 来运行程序。

你可能想知道，当 `learngo` 目录包含多个 Go 文件而不只是只有 `main.go` 时会发生什么。在这种情况下，`go install` 将如何工作？ 请继续往下看，我们将在了解软件包和 Go 模块时讨论这些内容。

### 2. go build

运行程序的第二个选项是使用 `go build`。`go build` 与 `go install` 非常相似，不同之处在于它不会将编译的二进制文件安装（拷贝）到路径 `~/go/bin/`，而是在 `go build` 所在的文件夹下创建二进制文件：

在命令行输入以下命令来切换当前目录到 `learngo`：

```sh
cd ~/Desktop/learngo/
```

然后输入下面的命令：

```sh
go build
```

上面的命令将会在当前目录下创建一个名为 `learngo` 的二进制可执行文件。`ls -al` 命令可以证实名为 `learngo` 的文件被创建了。

输入 `./learngo` 来运行程序，将会输入和前面一样的内容：


```sh
Hello World
```

到此，我们用 `go build` 也成功地运行了我们的第一个 Go 程序 😁

### 3. go run

第三个运行程序的方法是使用 `go run` 命令。

在命令行输入 `cd ~/Desktop/learngo` 命令来改变当前目录为 `learngo`。

然后输入以下命令。

```sh
go run main.go
```

输入以上命令后，我们也可以看到一样的输出：

```sh
Hello World
```

`go run` 命令和 `go build` 或 `go install` 命令之间的一个细微差别是，`go run` 要求使用 `.go` 文件的名称作为参数。

在引擎盖下，`go run` 的工作原理与 `go build` 非常相似。无需将程序编译并安装到当前目录，而是将文件编译到一个临时位置并从该位置运行文件。如果你想知道 `go run` 将文件编译到的位置，请使用 `--work` 参数运行 `go run`。

```sh
go run --work main.go
```

在我的场景中，运行以上命令会输出下面的内容：

```sh
WORK=/var/folders/mf/_fk8g5jn23gcw970pypqlv4m0000gn/T/go-build3519209434
Hello World
```

`WORK` 的值表示程序将被编译到的一个临时位置。

就我的场景而言，程序被编译到 `/var/folders/mf/_fk8g5jn23gcw970pypqlv4m0000gn/T/go-build3519209434` 。这可能因你的情况而异 😁

### 4. Go Playground

运行程序的最后一种方法是使用 go playground。尽管此方法有一些限制，但由于我们可以使用浏览器并且不需要在本地本地安装 Go：我已经为 Hello World 程序创建了一个 playground。 [点击此处](https://play.golang.org/p/oXGayDtoLPh) 以在线运行该程序。

你还可以使用 Go Playground 与他人分享你的源代码。

既然我们知道4种不同的方式来运行程序，那么你可能会很困惑该使用哪种方法。答案是，当我想快速检查逻辑或找出标准库函数如何工作时，通常使用 [playground](https://play.golang.org/)。在大多数其他情况下，我更喜欢 `go install`，因为它为我提供了从终端中任何目录运行程序的选项，因为它将所有程序编译到标准的 `~/go/bin/` 路径。

## 对 Hello World 程序的简短解析

这是我们刚刚创建的简单的 hello world 程序：

```go
package main

import "fmt"

func main() {
  fmt.Println("Hello World")
}
```

我们将简要讨论该程序的每一行的作用。在接下来的教程中，我们将深入研究程序的每个部分。

**package main** - 每个 go 文件都必须以 `package name` 开始。Packages 用于提供代码分隔和可重用性。此处使用包名称 `main`。主要功能应始终保留在 main package 中。

**import "fmt"** - `import` 语句用于导入其他软件包。在我们的例子中，`fmt` 包被导入，它将在 `main` 函数中用于将文本打印到标准输出。

**func main()** - `func` 关键字标记函数的开始。`main` 是一个特殊函数。程序从 `main` 函数开始执行。大括号 `{` 和 `}` 表示 `main` 函数的开始和结束。

**fmt.Println("Hello World")** - `fmt` 软件包的 `PrintIn` 函数用于将文本写入标准输出。`package.function()` 是在包中调用函数的语法。

本文的实例代码可以在 [github](https://github.com/golangbot/hello) 下载。

> 原文地址 [Hello World](https://golangbot.com/hello-world-gomod/)
> 原文作者：[Naveen Ramanathan](https://golangbot.com/about/)
> 译文出自：[紫升翻译计划](https://blog.zisheng.pro/categories/%E6%B4%9B%E7%AB%B9%E7%BF%BB%E8%AF%91%E8%AE%A1%E5%88%92/)
