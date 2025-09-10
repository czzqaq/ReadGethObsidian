
# 说明

从 L2 execution engine 开始构造block 开始，读取链信息，以及正在处理的block（作为 payload），加入到系统的L2 共识链上。

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

build 区块的主要动作，把payload 变为一个Block。Payload 来自L2 Engine，单纯是把Engine 中的处理中的区块拿出来。这里的L2 Engine 不是Optimism 的实现了。

成功执行后 build sealed
## build sealed

开始一个 `PayloadProcessEvent`  

## payload_process

先对payload 做了一个检查，时间和head.number 的检查，然后把payload 对应的block 添加到L2 的链（`OracleBackedL2Chain`) 中。
如果一切顺利，进入：`PayloadSuccessEvent`


## payload_success
这个payload 变成的block(来源是 build seal) 被作为一个 unsafe block 被设置。
如果它直接来源于 L1，那么认为是pending safe。

