
# DerivationPipeline

 描述了 derive 的流程

L1Traversal → L1Retrieval → FrameQueue → ChannelMux → 
ChannelInReader → BatchMux → AttributesQueue


## reset
从后往前执行一遍
```
stages := []ResettableStage{ l1Traversal, l1Src, altDA, frameQueue, channelMux, chInReader, batchMux, attributesQueue }
```

## step
```go
func (dp *DerivationPipeline) Step(ctx context.Context, pendingSafeHead eth.L2BlockRef)
```
核心，前进到下一个区块
1. 如果没有完成reset，则继续reset
2. 对比 prevOrigin 和 attrib.Origin()，其中，origin: 当前派生的 L1 原点（该点之前的 L1 已决定了当前的 L2 safe 链）.如果不一致，更新origin的指向，确保L1的连续性。 见[[#transformStages]]
3. 从 AttributesQueue 获取待处理的区块属性，获取下一个属性。存在属性不足，derive 失败的情况。


## transformStages

是“跨 fork 的热切换钩子”，所谓fork，就是Ecotone、Delta、Canyon等等 版本，如果版本不对齐了，就要把L2的规则和状态移动到和新版本对齐。
总之L2 也有版本代号，L1 变了L2也要跟着变变。
>Holocene 是 Optimism OP Stack 的一次协议升级（fork）代号。和以太坊用“伦敦/上海/坎昆”命名升级类似，OP Stack 用“Bedrock → … → Holocene”等代号标识不同阶段的协议规则与数据格式变更。
 
# 相关
## deriver.go
pipeline 的实际调用者. 事件驱动，例如 PipelineStepEvent、ForceResetEvent 等。只是一个包装。
详见 [[deriver.go]]
