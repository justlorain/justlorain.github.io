---
title: "PHOS: A Go channel extension with internal handlers"
date: 2023-05-05T00:26:25+08:00
draft: false
tags: ["go", "opensource", "programming"]
categories: ["BINARY WEB ECO"]
featuredImage: "/posts/phos/phos.png"
featuredImagePreview: "/posts/phos/phos.png"
---

## Introduction

Channels are an essential feature of Golang. Skilled use of channels can make concurrent programming feel natural. This article introduces a channel extension [PHOS](https://github.com/B1NARY-GR0UP/phos) based on Golang native channels. PHOS allows us to perform specific processing on the data sent to the channel. PHOS also has rich configuration options that can be customized according to different requirements.

## What is PHOS?

PHOS is a Golang channel extension with internal handlers and diversified options.

## Installation

> Note: Before installation, make sure your Golang version supports generics (Go >= 1.18).

Execute the following command to install PHOS:

```shell
go get github.com/B1NARY-GR0UP/phos
```

## Quick Start

In the following example, we first define our own built-in processor. Then we create a PHOS instance, add the processor to PHOS's processor chain, and finally, we can get the result processed by the processor by passing in the data and print it out. That is, `BINARY += -PHOS => BINARY-PHOS`.

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

## User Guide

### About Creation

PHOS is based on generics. When creating an instance through the `phos.New` method, we can set the generics to process data of a specific type.

When creating an instance, you don't need to specify the buffer size of the channel, as you would with a regular channel. You can assume that PHOS has an unlimited buffer.

PHOS also supports many custom options. We can append these options when creating an instance through `phos.New`. For example:

```go
ph := phos.New[string](phos.WithZero(), phos.WithTimeout(time.Second*5))
```

More supported options will be listed in tables in later chapters.

### About Input and Output

PHOS implements data input and output using a read-only and a write-only channel.

- **Input**

  After creating an instance through `phos.New`, you can input data to PHOS using the public write-only channel `In`. In the quick start example, we passed in a string to `In`. Note that the type of the data passed in should be consistent with the generic type used when creating the instance. As we mentioned earlier, you can input data to PHOS almost without limits without specifying the size of the buffer:

  ```go
  ph.In <- "BINARY"
  ph.In <- "Hello"
  ph.In <- "Foo"
  ...
  ```

- **Output**

  We can also read the data processed by PHOS through the public read-only channel `Out`. It should be noted that the output value of `Out` is encapsulated as a `Result` type:

    - The `Data` field is the processed result, and its type also matches the generic type of the instance;

    - The `OK` field is consistent with the second optional variable usage when we usually get data from the channel, which can be used to determine whether data can be read from the channel;

  > Note: When using PHOS, you should not use the second return value of the Out channel but use the OK field of the Result, because the second return value of the Out channel will always be true, and its erroneous use will cause bugs.

    - The `Err` field will prompt some exceptions that may occur when using PHOS, such as timeout exceptions, internal processor processing exceptions, which will be explained in the following text.

   ```go
   // Result PHOS output result
   type Result[T any] struct {
   	Data T
   	// Note: You should use the OK of Result rather than the second return value of PHOS Out channel
   	OK  bool
   	Err *Error
   }
   ```


### About internal handlers

Built-in processors are one of the most important features of PHOS. By setting one or more built-in processors, PHOS will execute all built-in processors for each input data **in order**, and we can pass the result processed by the previous processor to the next processor for further processing through the form of return values.

A valid processor function signature is as follows:

```go
 func(ctx context.Context, data T) (T, error)
```

We can construct a processor chain with PHOS's `Handlers` public slice field by appending our own custom processors. For example, we have constructed a processor chain with three processors here. Each processor will perform an addition operation on the input data once, and finally, each input data will be returned to us after being added by three (+1 * 3):

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

### About error handling

PHOS wraps and classifies exceptions, mainly into three types:

```go
 const (
 	_ ErrorType = iota
 	TimeoutErr
 	HandleErr
 	CtxErr
 )
```

- **Timeout exception**

  The timeout mechanism is a core feature of PHOS. It specifies the maximum processing time for each data in the built-in processor chain. If a timeout occurs, a timeout exception will be triggered. If the timeout exception handler option is not configured, the corresponding initial data will be output, and the timeout exception will be displayed in the `Err` field of `Result`; if the timeout exception handler is configured, the data processed by the timeout exception handler will be output.

- **Internal handlers exception**

  If an exception occurs in any of the Internal handlers in the Internal handlers chain, the processor chain will terminate. Whether to return the last processed data or the data processed by the exception handler depends on whether the Internal handlers exception handler is configured. Similarly, the exception will be displayed in the `Err` field of `Result`.

- **Context Exception**

  Context exceptions occur when the `WithContext` option is configured and triggered. At this time, the data processing will be terminated immediately, and whether to return the initial data or the data processed by the exception handler depends on whether the `ErrDoneFunc` is configured. The exception information will also be displayed in the `Err` field of `Result`.

- **Zero Value Option**

  When the zero value option is enabled, a zero value of the corresponding data type will be returned whenever any of the above exceptions occur. This will overwrite the data processing results of the configured timeout exception handler and built-in processor exception handler.

## Configuration

| Option               | Default                | Description                                                                          |
|----------------------|------------------------|--------------------------------------------------------------------------------------|
| `WithContext`        | `context.Background()` | Set context for PHOS                                                                 |
| `WithZero`           | `false`                | Set zero value for return when error happened                                        |
| `WithTimeout`        | `time.Second * 3`      | Set timeout for handlers execution                                                   |
| `WithErrHandleFunc`  | `nil`                  | Set error handle function for PHOS which will be called when handle error happened   |
| `WithErrTimeoutFunc` | `nil`                  | Set error timeout function for PHOS which will be called when timeout error happened |
| `WithErrDoneFunc`    | `nil`                  | Set err done function for PHOS which will be called when context done happened       |

## Summary

The above is an introduction to [PHOS](https://github.com/B1NARY-GR0UP/phos), a Golang channel extension with internal handlers and diversified options. It is hoped that it can provide you with some help or ideas for your development. If there are any mistakes or issues, please feel free to leave a comment or send a private message.

The next article will elaborate on the design and implementation of PHOS. If you are interested in PHOS, please give it a Star on [GitHub](https://github.com/B1NARY-GR0UP/phos).

## Reference List

- https://github.com/B1NARY-GR0UP/phos


