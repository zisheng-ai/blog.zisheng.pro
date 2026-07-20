---
title: Go 语言系列教程の字符串
date: 2023-03-19 15:14:00
cover: https://cdn.jsdelivr.net/gh/youngjuning/images@main/1679280067130.png
description: 在 Go 中，一个字符串是字节的一个切片。字符串可以通过将一组字符放在双引号内来创建
categories:
  - [Go]
  - [紫升翻译计划]
tags:
  - Go
  - Go 教程
  - 紫升翻译计划
  - Go 字符串
  - Go String
---

在 Go 中，String 值得特别一提，因为与其他语言相比，它们在实现上有所不同。

## String 是什么？

**在 Go 中，一个字符串是字节的一个切片。字符串可以通过将一组字符放在双引号内来创建**

让我们看看一个简单的例子，创建一个 `string` 并打印出来。

```go
package main

import (
    "fmt"
)

func main() {
    name := "Hello World"
    fmt.Println(name)
}
```

[Run in playground](https://play.golang.org/p/o9OVDgEMU0)

上述程序将打印 `Hello World`。

Go 中的字符串是 [符合 Unicode 标准](https://naveenr.net/unicode-character-set-and-utf-8-utf-16-utf-32-encoding/) 并且是 [UTF-8 编码](https://naveenr.net/unicode-character-set-and-utf-8-utf-16-utf-32-encoding/) 的。

## 访问一个字符串的单个字节

由于字符串是字节的一个切片，所以可以访问字符串的每个字节。

```go
package main

import (
    "fmt"
)

func printBytes(s string) {
    fmt.Printf("Bytes: ")
    for i := 0; i < len(s); i++ {
      fmt.Printf("%x ", s[i])
    }
}

func main() {
  name := "Hello World"
    fmt.Printf("String: %s\n", name) // 输入的字符串被打印出来
    printBytes(name)
}
```

[Run in playground](https://play.golang.org/p/B3KgBBQhiN9)

`%s` 是用于打印字符串的格式化标识符。`len(s)` 返回字符串中的字节数，我们使用 `for` 循环以十六进制符号打印这些字节。`%x` 是十六进制的格式指定符。上述程序的输出结果是：

```sh
String: Hello World
Bytes: 48 65 6c 6c 6f 20 57 6f 72 6c 64
```

这是 `Hello World` 的 [Unicode UT8 编码](<https://mothereff.in/utf-8#Hello World>) 值. 为了更好地理解字符串，需要对 Unicode 和 UTF-8 有一个基本的了解。 我推荐阅读 https://naveenr.net/unicode-character-set-and-utf-8-utf-16-utf-32-encoding/ 了解更多 Unicode 和 UTF-8 的知识。

## 访问字符串的单个字符

让我们对上述程序稍作修改，以打印字符串的字符。

```go
package main

import (
    "fmt"
)

func printBytes(s string) {
    fmt.Printf("Bytes: ")
    for i := 0; i < len(s); i++ {
        fmt.Printf("%x ", s[i])
    }
}

func printChars(s string) {
    fmt.Printf("Characters: ")
    for i := 0; i < len(s); i++ {
        fmt.Printf("%c ", s[i])
    }
}

func main() {
    name := "Hello World"
    fmt.Printf("String: %s\n", name)
    printChars(name)
    fmt.Printf("\n")
    printBytes(name)
}
```

[Run in playground](https://play.golang.org/p/ZkXmyVNsqv7)

`%c` 格式化标识符用于打印 `printChars` 方法中字符串参数中的字符。该程序打印的是：

```
String: Hello World
Characters: H e l l o   W o r l d
Bytes: 48 65 6c 6c 6f 20 57 6f 72 6c 64
```

虽然上面的程序看起来是访问字符串的单个字符的合法方式，但这有一个严重的错误。让我们来看看这个错误是什么。

```go
package main

import (
    "fmt"
)

func printBytes(s string) {
    fmt.Printf("Bytes: ")
    for i := 0; i < len(s); i++ {
        fmt.Printf("%x ", s[i])
    }
}

func printChars(s string) {
    fmt.Printf("Characters: ")
    for i := 0; i < len(s); i++ {
        fmt.Printf("%c ", s[i])
    }
}

func main() {
    name := "Hello World"
    fmt.Printf("String: %s\n", name)
    printChars(name)
    fmt.Printf("\n")
    printBytes(name)
    fmt.Printf("\n\n")
    name = "Señor"
    fmt.Printf("String: %s\n", name)
    printChars(name) //
    fmt.Printf("\n")
    printBytes(name)
}
```

[Run in playground](https://play.golang.org/p/2hyVf8l9fiO)

上述程序的输出是

```
String: Hello World
Characters: H e l l o   W o r l d
Bytes: 48 65 6c 6c 6f 20 57 6f 72 6c 64

String: Señor
Characters: S e Ã ± o r
Bytes: 53 65 c3 b1 6f 72
```

我们试图打印 **Señor** 的字符，但它输出 **S e Ã ± o r**，这是错误的。为什么这个程序对 `Señor` 会出错，而对 `Hello World ` 却能完全正常工作。原因是 `ñ` 的 Unicode 码位是 `U+00F1`，其 [UTF-8编码](https://mothereff.in/utf-8#%C3%B1) 占用了 2 个字节 `c3` 和 `b1`。我们试图打印字符，假设每个代码点是一个字节，这是错误的。**在 UTF-8 编码中，一个代码点可以占用 1个以上的字节。**那么我们如何解决这个问题？这就需要 **rune** 拯救我们的地方了。

## Rune

Rune 是 Go 中的一个内置类型，它是 `int32` 的别名。Rune 在 Go 中代表一个 Unicode 代码点。不管这个代码点占用多少字节，它都可以用 Rune 来表示。让我们修改上面的程序，用 Rune 来打印字符。

```go
package main

import (
    "fmt"
)

func printBytes(s string) {
    fmt.Printf("Bytes: ")
    for i := 0; i < len(s); i++ {
        fmt.Printf("%x ", s[i])
    }
}

func printChars(s string) {
    fmt.Printf("Characters: ")
    runes := []rune(s) // 字符串被转换为 runes 的切片
    // 然后我们对其进行循环，并显示这些字符。
    for i := 0; i < len(runes); i++ {
        fmt.Printf("%c ", runes[i])
    }
}

func main() {
    name := "Hello World"
    fmt.Printf("String: %s\n", name)
    printChars(name)
    fmt.Printf("\n")
    printBytes(name)
    fmt.Printf("\n\n")
    name = "Señor"
    fmt.Printf("String: %s\n", name)
    printChars(name)
    fmt.Printf("\n")
    printBytes(name)
}
```

[Run in playground](https://play.golang.org/p/n8rsfagm2SJ)

上述程序打印出：

```sh
String: Hello World
Characters: H e l l o   W o r l d
Bytes: 48 65 6c 6c 6f 20 57 6f 72 6c 64

String: Señor
Characters: S e ñ o r
Bytes: 53 65 c3 b1 6f 72
```

上述输出是完美的。只是我们想要的😀。

## 使用 `for range` 循环访问单个 Rune

上面的程序是一个完美的方式来迭代一个字符串的各个 Rune。但是 Go 为我们提供了一种更简单的方法，即使用 `for range` 循环来实现这一目的。

```go
package main

import (
    "fmt"
)

func charsAndBytePosition(s string) {
    // 使用 for range 循环迭代 string
    for index, rune := range s {
        fmt.Printf("%c starts at byte %d\n", rune, index)
    }
}

func main() {
    name := "Señor"
    charsAndBytePosition(name)
}
```

[Run in playground](https://play.golang.org/p/0ldNBeffjYI)

循环返回 Rune 开始的字节的位置，同时返回 Rune 的位置。这个程序输出：

```
S starts at byte 0
e starts at byte 1
ñ starts at byte 2
o starts at byte 4
r starts at byte 5
```

从上面的输出可以看出，`ñ` 占用了 2 个字节，因为下一个字符 `o` 是从第 4 字节开始的，而不是第 3 字节😀。

## 从一个字节片中创建一个字符串

```go
package main

import (
    "fmt"
)

func main() {
    byteSlice := []byte{0x43, 0x61, 0x66, 0xC3, 0xA9}
    str := string(byteSlice)
    fmt.Println(str)
}
```

[Run in playground](https://play.golang.org/p/Vr9pf8X8xO)

`byteSlice` 包含字符串 `Café`的 [UTF-8编码](https://mothereff.in/utf-8#Caf%C3%A9) 十六进制字节。该程序打印出

```
Café
```

如果我们有相当于十六进制的十进制值，怎么办？上面的程序能工作吗？让我们来看看。

```go
package main

import (
    "fmt"
)

func main() {
    byteSlice := []byte{67, 97, 102, 195, 169} // 十进制相当于 {'\x43', '\x61', '\x66', '\xC3', '\xA9'}
    str := string(byteSlice)
    fmt.Println(str)
}
```

[Run in playground](https://play.golang.org/p/jgsRowW6XN)

小数点值也可以，上述程序也会打印出 `Café`。

> 原文地址 [Golang tutorial series Strings](https://golangbot.com/strings/)
> 原文作者：[Naveen Ramanathan](https://golangbot.com/about/)
> 译文出自：[紫升翻译计划](https://blog.zisheng.pro/categories/%E6%B4%9B%E7%AB%B9%E7%BF%BB%E8%AF%91%E8%AE%A1%E5%88%92/)
