---
title: "DREAMEMO: 一个模块化，高扩展性，开箱即用的分布式缓存"
date: 2023-05-05T00:52:05+08:00
draft: false
tags: ["go", "opensource", "programming"]
categories: ["BINARY WEB ECO"]
featuredImage: "/posts/dreamemo/dreamemo.png"
featuredImagePreview: "/posts/dreamemo/dreamemo.png"
---

## 简介

如标题所示，[DREAMEMO](https://github.com/B1NARY-GR0UP/dreamemo) 是一个模块化，高扩展性，开箱即用的分布式缓存，参考了 [groupcache](https://github.com/golang/groupcache) 的实现，并对其重新进行架构，具体模块分化如下：

![arch](/posts/dreamemo/dreamemo-arch.png)

主要模块将在设计模块进行具体介绍。

## 快速开始

### 安装

执行以下命令安装 DREAMEMO：

```shell
go get github.com/B1NARY-GR0UP/dreamemo
```

### 以单节点模式运行

DREAMEMO 提供了使用默认配置的以 Standalone 模式运行的函数 `dream.StandAlone`，用户只需配置好对应数据源的 `source.Getter` 即可，使用示例如下：

```go
package main

import (
	"context"
	"fmt"
	"net/http"

	"github.com/B1NARY-GR0UP/dreamemo/common/constant"
	"github.com/B1NARY-GR0UP/dreamemo/dream"
	"github.com/B1NARY-GR0UP/dreamemo/guidance"
	"github.com/B1NARY-GR0UP/dreamemo/source"
	log "github.com/B1NARY-GR0UP/inquisitor/core"
	"github.com/B1NARY-GR0UP/piano/core"
	"github.com/B1NARY-GR0UP/piano/core/bin"
)

var db = map[string]string{
	"binary": "dreamemo",
	"hello":  "world",
	"ping":   "pong",
}

func getFromDB(_ context.Context, key string) ([]byte, error) {
	log.Info("Get from DB")
	if v, ok := db[key]; ok {
		return []byte(v), nil
	}
	return nil, fmt.Errorf("key %v is not exist", key)
}

// go run .
// curl localhost:8080/hello?key=ping
func main() {
	dream.StandAlone(source.GetterFunc(getFromDB))
	p := bin.Default(core.WithHostAddr(":8080"))
	p.GET("/hello", func(ctx context.Context, pk *core.PianoKey) {
		key := pk.Query("key")
		g := guidance.GetGroup(constant.DefaultGroupName)
		value, _ := g.Get(ctx, key)
		pk.JSON(http.StatusOK, core.M{
			key: value.String(),
		})
	})
	p.Play()
}
```

这里以 `map` 的形式模拟了数据库数据源，并使用 [PIANO](https://github.com/B1NARY-GR0UP/piano) HTTP 框架作为前端服务器，用户可以替换成其他 HTTP 框架例如 Hertz, Gin 等，通过 `go run .` 启动服务，接下来只需要访问对应 URL 即可获取 key 对应的值。

```shell
go run .
curl localhost:8080/hello?key=ping
```

### 以集群模式运行

DREAMEMO 同样提供了使用默认配置的以 Cluster 模式运行的函数 `dream.Cluster` ，用户只需配置好对应集群节点的地址以及数据源即可，使用示例如下：

```go
package main

import (
	"context"
	"fmt"
	"net/http"

	"github.com/B1NARY-GR0UP/dreamemo/common/constant"
	"github.com/B1NARY-GR0UP/dreamemo/common/util"
	"github.com/B1NARY-GR0UP/dreamemo/dream"
	"github.com/B1NARY-GR0UP/dreamemo/guidance"
	"github.com/B1NARY-GR0UP/dreamemo/source"
	log "github.com/B1NARY-GR0UP/inquisitor/core"
	"github.com/B1NARY-GR0UP/piano/core"
	"github.com/B1NARY-GR0UP/piano/core/bin"
)

var db = map[string]string{
	"binary": "dreamemo",
	"hello":  "world",
	"ping":   "pong",
}

func getFromDB(_ context.Context, key string) ([]byte, error) {
	log.Info("Get from DB")
	if v, ok := db[key]; ok {
		return []byte(v), nil
	}
	return nil, fmt.Errorf("key %v is not exist", key)
}

// go run . --addrs=http://localhost:7246,http://localhost:7247,http://localhost:7248 --api
// go run . --addrs=http://localhost:7247,http://localhost:7248,http://localhost:7246
// go run . --addrs=http://localhost:7248,http://localhost:7246,http://localhost:7247
// curl localhost:8080/hello?key=ping
func main() {
	addrs, api := util.ParseFlags()
	e := dream.Cluster(source.GetterFunc(getFromDB), addrs...)
	if api {
		go func() {
			p := bin.Default(core.WithHostAddr(":8080"))
			p.GET("/hello", func(ctx context.Context, pk *core.PianoKey) {
				key := pk.Query("key")
				g := guidance.GetGroup(constant.DefaultGroupName)
				value, _ := g.Get(ctx, key)
				pk.JSON(http.StatusOK, core.M{
					key: value.String(),
				})
			})
			p.Play()
		}()
	}
	e.Run()
}
```

这里同样以 `map` 模拟数据库数据源，并在 `localhost:7246`，`localhost:7247` 和 `localhost:7248` 配置了 3 个缓存节点，在 8080 端口配置了前端服务器，通过以下命令进行运行并访问对应 URL 即可获取 key 对应的值：

```shell
go run . --addrs=http://localhost:7246,http://localhost:7247,http://localhost:7248 --api
go run . --addrs=http://localhost:7247,http://localhost:7248,http://localhost:7246
go run . --addrs=http://localhost:7248,http://localhost:7246,http://localhost:7247
curl localhost:8080/hello?key=ping
```

### 自定义装配

前面介绍了使用 DREAMEMO 配置好的 `dream.StandAlone` 和 `dream.Cluster` 函数进行运行，这里将介绍以更加客制化的方式进行装配运行。

- **配置引擎**

  这里配置了引擎，`util.ParseFlags` 是提供的用于解析 flag 参数的工具方法，我们通过 `app.WithHostAddr` 配置引擎的监听地址并通过 `app.WithThrift0` 配置引擎使用 Thrift 作为序列化协议，并对其他节点进行注册。

  ```go
  addrs, api := util.ParseFlags()
  e := server.NewEngine(app.WithHostAddr(addrs[0]), app.WithThrift0())
  e.RegisterInstances(addrs...)
  ```

- **配置缓存淘汰策略**

  配置 LFU 策略

  ```go
  l := lfu.NewLFUCore()
  m := memo.NewMemo(l)
  ```

- **配置缓存组**

  这里通过 `guidance.WithGroupName` 配置缓存组名，通过 `guidance.WithThrift1` 配置使用 Thrift 作为序列化协议，注意这里需要和引擎保持一致，最后配置数据源 `source.Getter`

  ```go
  guidance.NewGroup(m, e, guidance.WithGroupName("hello"), guidance.WithThrift1(), guidance.WithGetter(source.GetterFunc(getFromDB)))
  ```

完整代码如下：

```go
package main

import (
	"context"
	"fmt"
	"net/http"

	"github.com/B1NARY-GR0UP/dreamemo/app"
	"github.com/B1NARY-GR0UP/dreamemo/app/server"
	"github.com/B1NARY-GR0UP/dreamemo/common/util"
	"github.com/B1NARY-GR0UP/dreamemo/guidance"
	"github.com/B1NARY-GR0UP/dreamemo/memo"
	"github.com/B1NARY-GR0UP/dreamemo/source"
	"github.com/B1NARY-GR0UP/dreamemo/strategy/eliminate/lfu"
	log "github.com/B1NARY-GR0UP/inquisitor/core"
	"github.com/B1NARY-GR0UP/piano/core"
	"github.com/B1NARY-GR0UP/piano/core/bin"
)

var db = map[string]string{
	"binary": "dreamemo",
	"hello":  "world",
	"ping":   "pong",
}

func getFromDB(_ context.Context, key string) ([]byte, error) {
	log.Info("Get from DB")
	if v, ok := db[key]; ok {
		return []byte(v), nil
	}
	return nil, fmt.Errorf("key %v is not exist", key)
}

// go run . --addrs=http://localhost:7246,http://localhost:7247,http://localhost:7248 --api
// go run . --addrs=http://localhost:7247,http://localhost:7248,http://localhost:7246
// go run . --addrs=http://localhost:7248,http://localhost:7246,http://localhost:7247
// curl localhost:8080/hello?key=ping
func main() {
	addrs, api := util.ParseFlags()
	e := server.NewEngine(app.WithHostAddr(addrs[0]), app.WithThrift0())
	e.RegisterInstances(addrs...)
	l := lfu.NewLFUCore()
	m := memo.NewMemo(l)
	guidance.NewGroup(m, e, guidance.WithGroupName("hello"), guidance.WithThrift1(), guidance.WithGetter(source.GetterFunc(getFromDB)))
	if api {
		go func() {
			p := bin.Default(core.WithHostAddr(":8080"))
			p.GET("/hello", func(ctx context.Context, pk *core.PianoKey) {
				key := pk.Query("key")
				g := guidance.GetGroup("hello")
				value, _ := g.Get(ctx, key)
				pk.JSON(http.StatusOK, core.M{
					key: value.String(),
				})
			})
			p.Play()
		}()
	}
	e.Run()
}
```

使用以下命令运行并获取缓存：

```shell
go run . --addrs=http://localhost:7246,http://localhost:7247,http://localhost:7248 --api
go run . --addrs=http://localhost:7247,http://localhost:7248,http://localhost:7246
go run . --addrs=http://localhost:7248,http://localhost:7246,http://localhost:7247
curl localhost:8080/hello?key=ping
```

## 设计

### 缓存淘汰

在缓存淘汰策略方面，DREAMEMO 支持 LRU 和 LFU 两种缓存淘汰算法，用户可以自行进行选择装配：

- 使用 LRU

  ```go
  c := lru.NewLRUCore()
  ```

- 使用 LFU

  ```go
  c := lfu.NewLFUCore()
  ```

同时 DREAMEMO 也提供了接口供用户扩展其他的淘汰策略，只需实现接口并将对象传入 `memo.NewMemo` 即可，接口如下：

```go
type ICore interface {
	Add(Key, Value)
	Get(Key) (Value, bool)
	Remove(Key)
	Clear()
	Name() string
}
```

### 序列化

DREAMEMO 支持 Thrift 和 Protobuf 两种序列化协议，默认使用 Protobuf 协议，如需使用 Thrift 协议，用户只需要在同时在配置引擎和缓存组处开启配置即可：

```go
e := server.NewEngine(app.WithThrift0()) // engine
guidance.NewGroup(m, e, guidance.WithThrift1()) // group
```

### 数据源

DREAMEMO 默认提供了以 Redis 作为数据源的配置：

```go
s := redis.NewSource()
```

同时也提供了接口供用户配置更多自己的数据源：

```
type Getter interface {
	Get(ctx context.Context, key string) ([]byte, error)
}
```

### 一致性哈希

DREAMEMO 和 groupcache 一样使用一致性哈希进行分布式节点选择，同时也提供了接口以供配置更多的策略。

## TODO

DREAMEMO 仍非常稚嫩，存在很多可供修改的点，之后将会提供更多可供的节点选择算法以及快速配置工具。

## 小结

以上就是分布式缓存 DREAMEMO 的介绍，希望可以给你带来一些帮助，项目地址在[这里](https://github.com/B1NARY-GR0UP/dreamemo)，欢迎 Star。如果有疑问可以在评论区指出，提 Issue 或者私信，以上。

## 参考列表

- https://github.com/B1NARY-GR0UP/dreamemo
- https://github.com/B1NARY-GR0UP/piano
- https://github.com/golang/groupcache
- https://github.com/geektutu/7days-golang

