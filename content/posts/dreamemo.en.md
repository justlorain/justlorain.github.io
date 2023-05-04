---
title: "DREAMEMO: An out-of-the-box, high-scalability, modular-design distributed cache"
date: 2023-05-05T00:52:11+08:00
draft: false
tags: ["go", "opensource", "programming"]
categories: ["BINARY WEB ECO"]
featuredImage: "/posts/dreamemo/dreamemo.png"
featuredImagePreview: "/posts/dreamemo/dreamemo.png"
---

## Introduction

As shown in the title, [DREAMEMO](https://github.com/B1NARY-GR0UP/dreamemo) is a distributed cache with out-of-the-box, high-scalability, modular-design features.The [groupcache](https://github.com/golang/groupcache) implementation is referenced, and re-structured, specific module differentiation is as follows:

![arch](/posts/dreamemo/dreamemo-arch.png)

The main modules will be introduced in detail in the design module.

## Quick Start

### Install

Execute following command to install DREAMEMO:

```shell
go get github.com/B1NARY-GR0UP/dreamemo
```

### Run with standalone mode

DREAMEMO provides a function `dream.StandAlone` used default configuration to help user to start in standalone mode swiftly. All you need to do is configure the `source.Getter` of the corresponding data source.

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

Here in the form of a ` map ` simulation data source database, and use [PIANO](https://github.com/B1NARY-GR0UP/piano) HTTP framework as a front-end server instead of using other HTTP frameworks like Hertz, Gin, etc. You can start the server with `go run .` and then simply visit the URL to retrieve the key.

```shell
go run .
curl localhost:8080/hello?key=ping
```

### Run with cluster mode

DREAMEMO also provides a `dream.Cluster` function that runs in Cluster mode using the default configuration. All you need to do is configure the address of the corresponding cluster node and the data source.

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

Again, the database data source is simulated as `map`, and three cache nodes are configured at `localhost:7246`, `localhost:7247` and `localhost:7248`, the frontend server is configured on port `8080`. To retrieve the value of the key, run the following command and visit the URL:

```shell
go run . --addrs=http://localhost:7246,http://localhost:7247,http://localhost:7248 --api
go run . --addrs=http://localhost:7247,http://localhost:7248,http://localhost:7246
go run . --addrs=http://localhost:7248,http://localhost:7246,http://localhost:7247
curl localhost:8080/hello?key=ping
```

### Custom Assemble

`dream.StandAlone` and `dream.Cluster` functions configured by DREAMEMO were used earlier, here we'll look at a more customized way of running the assembly.

- **Configure Engine**

  Here we configure the engine, `util.ParseFlags` is a utility method provided to parse flag arguments. We use `app.WithHostAddr` to configure the engine to listen to and `app.WithThrift0` to use Thrift as the serialization protocol and register other nodes.

  ```go
  addrs, api := util.ParseFlags()
  e := server.NewEngine(app.WithHostAddr(addrs[0]), app.WithThrift0())
  e.RegisterInstances(addrs...)
  ```

- **Configure cache eliminate strategy**

  Configure LFU

  ```go
  l := lfu.NewLFUCore()
  m := memo.NewMemo(l)
  ```

- **Configure cache group**

  We configure the cache group name with `guidance.WithGroupName`, configure Thrift as the serialization protocol with `guidance.WithThrift1`, be consistent with the engine, and configure the data source with `source.Getter`

  ```go
  guidance.NewGroup(m, e, guidance.WithGroupName("hello"), guidance.WithThrift1(), guidance.WithGetter(source.GetterFunc(getFromDB)))
  ```

The complete code is as follows:

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

Run and get the cache value with the following command:

```shell
go run . --addrs=http://localhost:7246,http://localhost:7247,http://localhost:7248 --api
go run . --addrs=http://localhost:7247,http://localhost:7248,http://localhost:7246
go run . --addrs=http://localhost:7248,http://localhost:7246,http://localhost:7247
curl localhost:8080/hello?key=ping
```

## Design

### Eliminate strategy

In terms of cache elimination strategy, DREAMEMO supports LRU and LFU cache elimination algorithms, and users can choose and assemble them by themselves:

- **LRU**

  ```go
  c := lru.NewLRUCore()
  ```

- **LFU**

  ```go
  c := lfu.NewLFUCore()
  ```

DREAMEMO also provides an interface for uses to extend other eliminate strategies. Just implement the interface and pass the object to `memo.NewMemo` as follows:

```go
type ICore interface {
	Add(Key, Value)
	Get(Key) (Value, bool)
	Remove(Key)
	Clear()
	Name() string
}
```

### Serialization

DREAMEMO supports both Thrift and Protobuf serialization protocols.The default is Protobuf, and to use Thrift, you need to enable Thrift in both the configuration engine and the cache group:

```go
e := server.NewEngine(app.WithThrift0()) // engine
guidance.NewGroup(m, e, guidance.WithThrift1()) // group
```

### Data Source

DREAMEMO provides a default configuration to use Redis as a data source:

```go
s := redis.NewSource()
```

DREAMEMO also provides an interface to configure more data sources of your own:

```
type Getter interface {
	Get(ctx context.Context, key string) ([]byte, error)
}
```

### Consistent Hash

DREAMEMO, like groupcache, uses consistent hashing for distributed node selection, while also providing an interface to configure more policies.

## TODO

DREAMEMO is still very young, and there are many changes that can be made to it. In the future, more node selection algorithms and quick configuration tools will be provided.

## Summary

That's all for this article. Hope this can help you. I would be appreciate if you could give [DREAMEMO](https://github.com/B1NARY-GR0UP/dreamemo) a star.

If you have any questions, please leave them in the comments or as issues. Thanks for reading.

## Reference List

- https://github.com/B1NARY-GR0UP/dreamemo
- https://github.com/B1NARY-GR0UP/piano
- https://github.com/golang/groupcache
- https://github.com/geektutu/7days-golang
