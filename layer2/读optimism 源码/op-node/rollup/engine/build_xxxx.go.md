
# 说明

## build start

它的触发通常是 rollup node 开始调用 **Execution Engine** 去构造一个新区块的开始时刻，比如 `SequencerActionEvent`
用来发送当前的链信息：
```go
    fcEvent := ForkchoiceUpdateEvent{
        UnsafeL2Head:    ev.Attributes.Parent,
        SafeL2Head:      eq.safeHead,
        FinalizedL2Head: eq.finalizedHead,
    }
```

同时开启一个 BuildStartedEvent，见下面的 build started

## build started
被build start 触发，表明build 确实已经开始了，而不是遭遇错误，发现链重组等。
开启一个BuildSealEvent ：
```go
BuildSealEvent{
            Info:         ev.Info,
            BuildStarted: ev.BuildStarted,
            Concluding:   ev.Concluding,
            DerivedFrom:  ev.DerivedFrom,
        }****
```

## build seal

build 区块的主要动作，把payload 变为一个Block。Payload 来自L2 Engine，单纯是把Engine 中的处理中的区块拿出来。
这里的L2 Engine 不是Optimism 的实现了。

