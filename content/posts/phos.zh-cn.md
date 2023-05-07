---
title: "PHOS: 一个具有内置处理器的 Go channel 扩展"
date: 2023-05-05T00:26:25+08:00
draft: false
tags: ["go", "opensource", "programming"]
categories: ["BINARY WEB ECO"]
featuredImage: "/posts/phos/phos.png"
featuredImagePreview: "/posts/phos/phos.png"
---

## 前言

Channel 是 Golang 一个非常重要的语言特性，熟练的使用 channel 可以让我们在并发编程中如鱼得水，本文将介绍一个基于 Golang 原生 channel 实现的 channel 扩展 [PHOS](https://github.com/B1NARY-GR0UP/phos)，通过使用 PHOS 可以让我们对传入 channel 的数据进行特定的处理，同时 PHOS 也具有非常丰富的配置选项，我们可以基于不同需求对 PHOS 进行定制使用。

## 什么是 PHOS ?

PHOS 是一个具有内置处理器和多样化选项的 Golang Channel 扩展。

## 安装

> 注意：在安装之前，确保你使用的 Golang 版本支持泛型（Go >= 1.18）

执行以下命令安装 PHOS：

```shell
go get github.com/B1NARY-GR0UP/phos
```

## 快速开始

在以下示例中，我们首先定义了一个我们自己的 PHOS 内置处理器，然后通过创建一个 PHOS 实例并将这个处理器加入到 PHOS 的处理器链中，最后我们通过传入一个数据即可得到经过处理器处理后的结果并将其打印出来，即 `BINARY += -PHOS => BINARY-PHOS`。

```go
package main

import (
    "context"
    "fmt"

    "github.com/B1NARY-GR0UP/phos"
)

func hello(_ context.Context, data string) (string, error) {
    return data + "-PHOS", nil
}

func main() {
    ph := phos.New[string]()
    defer close(ph.In)
    // Note: You should decide which handlers you want to use once, 
    // otherwise multiple changes to the Handlers slice may cause data race problem
    ph.Handlers = append(ph.Handlers, hello)
    ph.In <- "BINARY"
    res := <-ph.Out
    fmt.Println(res.Data)
}
```

## 使用帮助

### 关于创建

PHOS 是基于泛型的，我们在通过 `phos.New` 方法来创建实例时即可通过设置泛型来对特定类型的数据进行处理。

并且在创建实例时你不需要像使用普通的 Channel 那样指定 Channel 的缓冲区大小，你可以假定 PHOS 的缓冲区没有限制。

同时 PHOS 也支持了许多的自定义选项，我们可以在通过 `phos.New` 创建实例时以选项嵌入的模式进行追加，例如：

```go
ph := phos.New[string](phos.WithZero(), phos.WithTimeout(time.Second*5))
```

更多支持的选项将在之后的章节以表格的形式列出。

### 关于输入输出

PHOS 通过一个只读和一个只写的 Channel 来实现数据的输入输出。

- **输入**

  在通过 `phos.New` 创建出一个实例后，通过 `In` 这个公开的只写 Channel 即可向 PHOS 中传入数据，例如在快速开始中我们向 `In` 中传入了一个字符串，需要注意的是传入数据的类型应该与你创建实例时所使用的泛型类型保持一致，并且正如我们前面提到的，你可以接近无限制的向 PHOS 中传入数据而不要指定缓冲区的大小：

  ```go
  ph.In <- "BINARY"
  ph.In <- "Hello"
  ph.In <- "Foo"
  ...
  ```

- **输出**

  我们同样可以通过 `Out` 这个公开的只读 Channel 来读取经过 PHOS 处理后的数据，需要注意的是，`Out` 输出的值被封装为 `Result` 类型：

    -  `Data` 字段是处理后的结果，其类型也与实例的泛型类型相匹配；

    - `OK` 字段与我们平时从 Channel 中获取数据的第二个可选变量用法一致，可以用来判断是否可以从 Channel 读取到数据；

      > 注意：在使用 PHOS 时你不应该再使用 Out Channel 的第二个返回值而是使用 Result 的 OK 字段，因为 Out Channel 的第二个返回值将一直为 true，对其错误的使用将导致 bug。

    - `Err` 字段会提示一些在使用 PHOS 时会出现的异常，例如超时异常，内部处理器处理异常，这将在下文进行阐述。

  ```go
  // Result PHOS output result
  type Result[T any] struct {
  	Data T
  	// Note: You should use the OK of Result rather than the second return value of PHOS Out channel
  	OK  bool
  	Err *Error
  }
  ```

### 关于内置处理器

内置处理器是 PHOS 最重要的特性之一，通过设置一个或多个内置处理器，PHOS 将按照顺序对每一个输入的数据**依次**执行所有的内置处理器，并且我们可以通过返回值的形式将前一个处理器处理后的结果传入下一个处理器之中继续进行处理。

一个合法的处理器函数签名如下：

```go
func(ctx context.Context, data T) (T, error)
```

我们可以通过对 PHOS 的 `Handlers` 公开切片字段追加我们自定义的处理器来构造一个处理器链，例如如下我们构造了一个具有三个处理器的处理器链，每一个处理器将对传入的数据进行一次加一操作，最终每个传入的数据都会在加三（+1 * 3）后返回给我们：

```go
package main

import (
	"context"
	"fmt"
	
	"github.com/B1NARY-GR0UP/phos"
)

func plusOne(_ context.Context, data int) (int, error) {
	return data + 1, nil
}

func main() {
	ph := phos.New[int]()
	defer close(ph.In)
	ph.Handlers = append(ph.Handlers, plusOne, plusOne, plusOne)
	ph.In <- 1
	ph.In <- 2
	ph.In <- 3
	res1 := <-ph.Out
	res2 := <-ph.Out
	res3 := <-ph.Out
	fmt.Println(res1.Data) // 4
	fmt.Println(res2.Data) // 5
	fmt.Println(res3.Data) // 6
}
```

### 关于异常处理

PHOS 对异常进行了包装分类，主要分为三种类型：

```go
const (
	_ ErrorType = iota
	TimeoutErr
	HandleErr
	CtxErr
)
```

- **超时异常**

  超时机制是 PHOS 的一个核心特性，它规定了内置处理器链对每个数据的最大处理时间，如果超时则会触发超时异常，如果不配置超时异常处理器选项则会输出对应输入的初始数据，并在 `Result` 的 `Err` 字段显示超时异常；如果配置了超时异常处理器，则会输出经过超时异常处理器处理后的数据。

- **内置处理器异常**

  如果在执行内置处理器链中的任何一个处理器发生了异常，则处理器链终止，根据是否配置了内置处理器异常处理器来决定返回的数据是最后处理过的数据还是经过异常处理器处理后的数据，同样异常会配置在 `Result ` 的 `Err` 字段。

- **上下文异常**

  上下文异常发生在配置了 `WithContext` 选项后并触发后的情况，这时数据处理会立刻终止并根据是否配置了 `ErrDoneFunc` 返回最开始的数据还是经过异常处理器处理后的数据，同样会将异常信息展示在 `Result` 的 `Err` 字段。

- **零值选项**

  开启零值选项后，只要发生以上任何一种异常，都将返回对应数据类型的零值，这会对配置的超时异常处理器和内置处理器异常处理器的数据处理结果进行**覆盖**。

## 配置

| 选项                   | 默认值                    | 描述                    |
|----------------------|------------------------|-----------------------|
| `WithContext`        | `context.Background()` | 设置 PHOS 的 context     |
| `WithZero`           | `false`                | 开启后出现异常时将返回对应类型的零值    |
| `WithTimeout`        | `time.Second * 3`      | 设置内置处理器链的超时时间         |
| `WithErrHandleFunc`  | `nil`                  | 设置出现内置处理器异常时的异常处理函数   |
| `WithErrTimeoutFunc` | `nil`                  | 设置出现超时异常时的异常处理函数      |
| `WithErrDoneFunc`    | `nil`                  | 设置出现上下文 Done 时的异常处理函数 |

## 总结

以上就是对 [PHOS](https://github.com/B1NARY-GR0UP/phos)，一个具有内置处理器和多样化选项的 Golang Channel 扩展的介绍，希望他可以为你的开发提供帮助或者为你提供一些思路，如果哪里写错了或者有问题欢迎评论或者私聊指出ww

下一篇文章将对 PHOS 的设计与实现思路进行阐述，如果你对 [PHOS](https://github.com/B1NARY-GR0UP/phos) 感兴趣，欢迎 Star !!!

## 后记

由于初版的 PHOS 被发现在某些场景下会出现 data race 问题，详见 [这个 issue](https://github.com/B1NARY-GR0UP/phos/issues/1) ，目前已经修复，并且由于对部分 API 进行了修改发布了新的中版本。主要调整的 API 为用户添加处理器 Handler 部分的方式，新的版本提供了 `Append`，`Remove`，`Len` 等方法而禁止用户对 PHOS 的 Handler 切片直接进行修改。整体使用方式基本不变，详情可以参考 [PHOS](https://github.com/B1NARY-GR0UP/phos) Readme。

## 参考列表

- https://github.com/B1NARY-GR0UP/phos

