# 架构

## 调用关系

这几个组件是相互依赖调用的关系,在下面的代码中，注意new 的时候都把一个组件传了进去。
```go
    frameQueue := NewFrameQueue(log, rollupCfg, l1Src)

    channelMux := NewChannelMux(log, spec, frameQueue, metrics)

    chInReader := NewChannelInReader(rollupCfg, log, channelMux, metrics)

    batchMux := NewBatchMux(log, rollupCfg, chInReader, l2Source)
```

在本章，从 batchMux 开始看起，最后到frame


batch 使用的 chInReader，是作为了NextBatchProvider 的接口的实现者被使用的，具体到 这个接口：
```go
NextBatch(ctx context.Context) (Batch, error)
```
---

chInReader 的实现中，调用了 channelMux 的 `NextRawChannel`，作为 RawChannelProvider，这部分的具体实现见： ChannelBank(channel_bank.go) 。在 channelMux.reset 中，规定了接口的实现由 ChannelBank 完成，或者版本不同的话由其他策略。
注意定义了一个 RawChannelProvider 接口在类里，详见[[#类里接口]]

ChannelBank 中调用了 NextFrameProvider 的 `NextFrame` 方法。

---

FrameQueue 就是这个  NextFrameProvider，它调用了 NextDataProvider 的 `NextData` 方法，即L1 retrival。


## 概念

### frame

简单的说，就是data 序列化后的结果。一个Data 会序列化为几段，每段固定长度，还有些frame ID  的必要信息。

### channel






## 依赖
### L2Source


# go

## 类型断言

```go
c.RawChannelProvider.(*ChannelBank)
```
即：
```go
v, ok := x.(T)
```
```markdown
- `x`：是接口类型的变量（比如 `c.RawChannelProvider`）
- `T`：你想断言的具体类型（比如 `*ChannelBank`）
- `v`：如果断言成功，`v` 就是类型 `T` 的值
- `ok`：是一个 bool，**表示断言是否成功**
```


## 类里接口

```go
type ChannelMux struct {
...
    // embedded active stage

    RawChannelProvider

}
...

c.RawChannelProvider = NewChannelBank(c.log, c.spec, c.prev, c.m)
```

