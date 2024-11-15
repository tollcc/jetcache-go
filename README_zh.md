# jetcache-go
![banner](docs/banner.png)

<p>
<a href="https://github.com/mgtv-tech/jetcache-go/actions"><img src="https://github.com/mgtv-tech/jetcache-go/workflows/Go/badge.svg" alt="Build Status"></a>
<a href="https://codecov.io/gh/mgtv-tech/jetcache-go"><img src="https://codecov.io/gh/mgtv-tech/jetcache-go/master/graph/badge.svg" alt="codeCov"></a>
<a href="https://goreportcard.com/report/github.com/mgtv-tech/jetcache-go"><img src="https://goreportcard.com/badge/github.com/mgtv-tech/jetcache-go" alt="Go Repport Card"></a>
<a href="https://github.com/mgtv-tech/jetcache-go/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-green" alt="License"></a>
<a href="https://github.com/mgtv-tech/jetcache-go/releases"><img src="https://img.shields.io/github/release/mgtv-tech/jetcache-go" alt="Release"></a>
</p>

Translations: [English](README.md) | [简体中文](README_zh.md)

# 简介
[jetcache-go](https://github.com/mgtv-tech/jetcache-go)是基于[go-redis/cache](https://github.com/go-redis/cache)拓展的通用缓存访问框架。
实现了类似Java版[JetCache](https://github.com/alibaba/jetcache)的核心功能，包括：

- ✅ 二级缓存自由组合：本地缓存、分布式缓存、本地缓存+分布式缓存
- ✅ `Once`接口采用单飞(`singleflight`)模式，高并发且线程安全
- ✅ 默认采用[MsgPack](https://github.com/vmihailenco/msgpack)来编解码Value。可选[sonic](https://github.com/bytedance/sonic)、原生`json`
- ✅ 本地缓存默认实现了[Ristretto](https://github.com/dgraph-io/ristretto)和[FreeCache](https://github.com/coocood/freecache)
- ✅ 分布式缓存默认实现了[go-redis/v9](https://github.com/redis/go-redis)的适配器，你也可以自定义实现
- ✅ 可以自定义`errNotFound`，通过占位符替换，缓存空结果防止缓存穿透
- ✅ 支持开启分布式缓存异步刷新
- ✅ 指标采集，默认实现了通过日志打印各级缓存的统计指标（QPM、Hit、Miss、Query、QueryFail）
- ✅ 分布式缓存查询故障自动降级
- ✅ `MGet`接口支持`Load`函数。带分布缓存场景，采用`Pipeline`模式实现 (v1.1.0+)
- ✅ 支持拓展缓存更新后所有GO进程的本地缓存失效 (v1.1.1+)

| 特性             | eko/gocache | go-redis/cache | mgtv-tech/jetcache-go |
|----------------|-------------|----------------|-----------------------|
| 多级缓存           | Yes         | Yes            | Yes                   |
| 缓存旁路(loadable) | Yes         | Yes            | Yes                   |
| 泛型支持           | Yes         | No             | Yes                   |
| 单飞模式           | Yes         | Yes            | Yes                   |
| 缓存更新监听器        | No          | No             | Yes                   |
| 自动刷新           | No          | No             | Yes                   |
| 指标采集           | Yes         | Yes (simple)   | Yes                   |
| 缓存空对象          | No          | No             | Yes                   |
| 批量查询           | No          | No             | Yes                   |
| 稀疏列表缓存         | No          | No             | Yes                   |

# 安装
使用最新版本的jetcache-go，您可以在项目中导入该库：
```shell
go get github.com/mgtv-tech/jetcache-go
```

# 快速开始

```go
package cache_test

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"time"

	"github.com/mgtv-tech/jetcache-go"
	"github.com/mgtv-tech/jetcache-go/local"
	"github.com/mgtv-tech/jetcache-go/remote"
	"github.com/redis/go-redis/v9"
)

var errRecordNotFound = errors.New("mock gorm.errRecordNotFound")

type object struct {
	Str string
	Num int
}

func Example_basicUsage() {
	ring := redis.NewRing(&redis.RingOptions{
		Addrs: map[string]string{
			"localhost": ":6379",
		},
	})

	mycache := cache.New(cache.WithName("any"),
		cache.WithRemote(remote.NewGoRedisV9Adapter(ring)),
		cache.WithLocal(local.NewFreeCache(256*local.MB, time.Minute)),
		cache.WithErrNotFound(errRecordNotFound))

	ctx := context.TODO()
	key := "mykey:1"
	obj, _ := mockDBGetObject(1)
	if err := mycache.Set(ctx, key, cache.Value(obj), cache.TTL(time.Hour)); err != nil {
		panic(err)
	}

	var wanted object
	if err := mycache.Get(ctx, key, &wanted); err == nil {
		fmt.Println(wanted)
	}
	// Output: {mystring 42}

	mycache.Close()
}

func Example_advancedUsage() {
	ring := redis.NewRing(&redis.RingOptions{
		Addrs: map[string]string{
			"localhost": ":6379",
		},
	})

	mycache := cache.New(cache.WithName("any"),
		cache.WithRemote(remote.NewGoRedisV9Adapter(ring)),
		cache.WithLocal(local.NewFreeCache(256*local.MB, time.Minute)),
		cache.WithErrNotFound(errRecordNotFound),
		cache.WithRefreshDuration(time.Minute))

	ctx := context.TODO()
	key := "mykey:1"
	obj := new(object)
	if err := mycache.Once(ctx, key, cache.Value(obj), cache.TTL(time.Hour), cache.Refresh(true),
		cache.Do(func(ctx context.Context) (any, error) {
			return mockDBGetObject(1)
		})); err != nil {
		panic(err)
	}
	fmt.Println(obj)
	// Output: &{mystring 42}

	mycache.Close()
}

func Example_mGetUsage() {
	ring := redis.NewRing(&redis.RingOptions{
		Addrs: map[string]string{
			"localhost": ":6379",
		},
	})

	mycache := cache.New(cache.WithName("any"),
		cache.WithRemote(remote.NewGoRedisV9Adapter(ring)),
		cache.WithLocal(local.NewFreeCache(256*local.MB, time.Minute)),
		cache.WithErrNotFound(errRecordNotFound),
		cache.WithRemoteExpiry(time.Minute),
	)
	cacheT := cache.NewT[int, *object](mycache)

	ctx := context.TODO()
	key := "mget"
	ids := []int{1, 2, 3}

	ret := cacheT.MGet(ctx, key, ids, func(ctx context.Context, ids []int) (map[int]*object, error) {
		return mockDBMGetObject(ids)
	})

	var b bytes.Buffer
	for _, id := range ids {
		b.WriteString(fmt.Sprintf("%v", ret[id]))
	}
	fmt.Println(b.String())
	// Output: &{mystring 1}&{mystring 2}<nil>

	cacheT.Close()
}

func Example_syncLocalUsage() {
	ring := redis.NewRing(&redis.RingOptions{
		Addrs: map[string]string{
			"localhost": ":6379",
		},
	})

	sourceID := "12345678" // Unique identifier for this cache instance
	channelName := "syncLocalChannel"
	pubSub := ring.Subscribe(context.Background(), channelName)

	mycache := cache.New(cache.WithName("any"),
		cache.WithRemote(remote.NewGoRedisV9Adapter(ring)),
		cache.WithLocal(local.NewFreeCache(256*local.MB, time.Minute)),
		cache.WithErrNotFound(errRecordNotFound),
		cache.WithRemoteExpiry(time.Minute),
		cache.WithSourceId(sourceID),
		cache.WithSyncLocal(true),
		cache.WithEventHandler(func(event *cache.Event) {
			// Broadcast local cache invalidation for the received keys
			bs, _ := json.Marshal(event)
			ring.Publish(context.Background(), channelName, string(bs))
		}),
	)
	obj, _ := mockDBGetObject(1)
	if err := mycache.Set(context.TODO(), "mykey", cache.Value(obj), cache.TTL(time.Hour)); err != nil {
		panic(err)
	}

	go func() {
		for {
			msg := <-pubSub.Channel()
			var event *cache.Event
			if err := json.Unmarshal([]byte(msg.Payload), &event); err != nil {
				panic(err)
			}
			fmt.Println(event.Keys)

			// Invalidate local cache for received keys (except own events)
			if event.SourceID != sourceID {
				for _, key := range event.Keys {
					mycache.DeleteFromLocalCache(key)
				}
			}
		}
	}()

	// Output: [mykey]
	mycache.Close()
	time.Sleep(time.Second)
}

func mockDBGetObject(id int) (*object, error) {
	if id > 100 {
		return nil, errRecordNotFound
	}
	return &object{Str: "mystring", Num: 42}, nil
}

func mockDBMGetObject(ids []int) (map[int]*object, error) {
	ret := make(map[int]*object)
	for _, id := range ids {
		if id == 3 {
			continue
		}
		ret[id] = &object{Str: "mystring", Num: id}
	}
	return ret, nil
}
```

## 配置选项
```go
// Options 用于存储缓存选项。
type Options struct {
name                       string             // 缓存名称，用于日志标识和指标报告
remote                     remote.Remote      // remote 是分布式缓存，例如 Redis。
local                      local.Local        // local 是内存缓存，例如 FreeCache。
codec                      string             // value的编码和解码方法。默认为 "msgpack.Name"。也可以自定义。
errNotFound                error              // 缓存未命中时返回的错误。用于防止缓存穿透（即缓存空对象）。
remoteExpiry               time.Duration      // 远程缓存 TTL，默认为 1 小时。
notFoundExpiry             time.Duration      // 缓存未命中时占位符缓存的过期时间。默认为 1 分钟。
offset                     time.Duration      // 缓存未命中时的过期时间抖动因子。
refreshDuration            time.Duration      // 异步缓存刷新的间隔。默认为 0（禁用刷新）。
stopRefreshAfterLastAccess time.Duration      // 缓存停止刷新之前的持续时间（上次访问后）。默认为 refreshDuration + 1 秒。
refreshConcurrency         int                // 并发缓存刷新的最大数量。默认为 4。
statsDisabled              bool               // 禁用缓存统计的标志。
statsHandler               stats.Handler      // 指标统计收集器。
sourceID                   string             // 缓存实例的唯一标识符。
syncLocal                  bool               // 启用同步本地缓存的事件（仅适用于 "Both" 缓存类型）。
eventChBufSize             int                // 事件通道的缓冲区大小（默认为 100）。
eventHandler               func(event *Event) // 处理本地缓存失效事件的函数。
}
```

## 缓存指标收集和统计
您可以实现`stats.Handler`接口并注册到Cache组件来自定义收集指标，例如使用[Prometheus](https://github.com/prometheus/client_golang)
采集指标。我们默认实现了通过日志打印统计指标，如下所示：
```shell
2023/09/11 16:42:30.695294 statslogger.go:178: [INFO] jetcache-go stats last 1m0s.
cache       |         qpm|   hit_ratio|         hit|        miss|       query|  query_fail
------------+------------+------------+------------+------------+------------+------------
bench       |   216440123|     100.00%|   216439867|         256|         256|           0
bench_local |   216440123|     100.00%|   216434970|        5153|           -|           -
bench_remote|        5153|      95.03%|        4897|         256|           -|           -
------------+------------+------------+------------+------------+------------+------------
```

## 自定义日志
```go
import "github.com/mgtv-tech/jetcache-go/logger"

// Set your Logger
logger.SetDefaultLogger(l logger.Logger)
```

## 自定义编解码
```go
import (
    "github.com/mgtv-tech/jetcache-go"
    "github.com/mgtv-tech/jetcache-go/encoding"
)

// Register your codec
encoding.RegisterCodec(codec Codec)

// Set your codec name
mycache := cache.New[string, any]("any",
    cache.WithRemote(...),
    cache.WithCodec(yourCodecName string))
```

# 使用场景说明

## 自动刷新缓存
`jetcache-go`提供了自动刷新缓存的能力，目的是为了防止缓存失效时造成的雪崩效应打爆数据库。对一些key比较少，实时性要求不高，加载开销非常大的缓存场景，适合使用自动刷新。下面的代码指定每分钟刷新一次，1小时如果没有访问就停止刷新。如果缓存是redis或者多级缓存最后一级是redis，缓存加载行为是全局唯一的，也就是说不管有多少台服务器，同时只有一个服务器在刷新，目的是为了降低后端的加载负担。
```go
mycache := cache.New(cache.WithName("any"),
		// ...
		// cache.WithRefreshDuration 设置异步刷新时间间隔
		cache.WithRefreshDuration(time.Minute),
		// cache.WithStopRefreshAfterLastAccess 设置缓存 key 没有访问后的刷新任务取消时间
        cache.WithStopRefreshAfterLastAccess(time.Hour))

// `Once` 接口通过 `cache.Refresh(true)` 开启自动刷新
err := mycache.Once(ctx, key, cache.Value(obj), cache.Refresh(true), cache.Do(func(ctx context.Context) (any, error) {
    return mockDBGetObject(1)
}))
```

## MGet批量查询
`MGet` 通过 `golang`的泛型机制 + `Load` 函数，非常友好的多级缓存批量查询ID对应的实体。如果缓存是redis或者多级缓存最后一级是redis，查询时采用`Pipeline`实现读写操作，提升性能。需要说明是，针对异常场景（IO异常、序列化异常等），我们设计思路是尽可能提供有损服务，防止穿透。
```go
mycache := cache.New(cache.WithName("any"),
		// ...
		cache.WithRemoteExpiry(time.Minute),
	)
cacheT := cache.NewT[int, *object](mycache)

ctx := context.TODO()
key := "mykey"
ids := []int{1, 2, 3}

ret := cacheT.MGet(ctx, key, ids, func(ctx context.Context, ids []int) (map[int]*object, error) {
    return mockDBMGetObject(ids)
})
```

## Codec编解码选择
`jetcache-go`默认实现了三种编解码方式，[sonic](https://github.com/bytedance/sonic)、[MsgPack](https://github.com/vmihailenco/msgpack)和原生`json`。

**选择指导：**

- **追求编解码性能：** 例如本地缓存命中率极高，但本地缓存byte数组转对象的反序列化操作非常耗CPU，那么选择`sonic`。
- **兼顾性能和极致的存储空间：** 选择`MsgPack`，MsgPack采用MsgPack编解码，内容>64个字节，会采用`snappy`压缩。

> Tip：使用的时候记得按需导包来完成对应的编解码器注册
```go
 _ "github.com/mgtv-tech/jetcache-go/encoding/sonic"
```

# 插件

插件项目：[jetcache-go-plugin](https://github.com/mgtv-tech/jetcache-go-plugin)，欢迎参与共建。目前已实现如下插件：

- [Remote Adapter using go-redis v8](https://github.com/mgtv-tech/jetcache-go-plugin?tab=readme-ov-file#redisgo-redis-v8)
- [Prometheus-based Stats Handler](https://github.com/mgtv-tech/jetcache-go-plugin?tab=readme-ov-file#prometheus)

# 贡献

欢迎大家一起来完善jetcache-go。如果您有任何疑问、建议或者想添加其他功能，请直接提交issue或者PR。

请按照以下步骤来提交PR：

- 克隆仓库
- 创建一个新分支：如果是新功能分支，则命名为feature-xxx；如果是修复bug，则命名为bug-xxx
- 在PR中详细描述更改的内容

# 联系

如果您有任何问题，请联系 `daoshenzzg@gmail.com`。

