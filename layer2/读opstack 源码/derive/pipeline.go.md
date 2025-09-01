
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

## l1TraversalStage
L1Traversal 的实现。
从当前的 L1 区块出发，尝试获取下一个 L1 区块的元信息（Header + Receipts）。

详见 [[l1_traversal.go]]

## L1Retrieval
它从 L1 区块中提取出 **Rollup 相关的数据**，供后续模块处理。
详见 [[l1_traversal.go]]

## Frame & Channel
L1Retrieval（获取 L1 数据） ──▶ FrameQueue（解析为 Frame） ──▶ ChannelMux（组装完整 Channel）-> channelInReader(将channel 读入batch)

在 Optimism 的 batch 数据结构中，L2 的交易数据被拆分为多个 **通道（channel）**，每个通道又被拆成多个 **帧（frame）**。

详见 [[frame and channel]]


## batchMux & attributesQueue

batchMux 是一个简单的batch queue 缓冲。

attributesQueue 将已解码的 **L2 Batch** 转换为 **`PayloadAttributes`**
其中：

```go
type PayloadAttributes struct {
	// value for the timestamp field of the new payload
	Timestamp Uint64Quantity `json:"timestamp"`
	// value for the random field of the new payload
	PrevRandao Bytes32 `json:"prevRandao"`
	// suggested value for the coinbase field of the new payload
	SuggestedFeeRecipient common.Address `json:"suggestedFeeRecipient"`
	// Withdrawals to include into the block -- should be nil or empty depending on Shanghai enablement
	Withdrawals *types.Withdrawals `json:"withdrawals,omitempty"`
	// parentBeaconBlockRoot optional extension in Dencun
	ParentBeaconBlockRoot *common.Hash `json:"parentBeaconBlockRoot,omitempty"`

	// Optimism additions

	// Transactions to force into the block (always at the start of the transactions list).
	Transactions []Data `json:"transactions,omitempty"`
	// NoTxPool to disable adding any transactions from the transaction-pool.
	NoTxPool bool `json:"noTxPool,omitempty"`
	// GasLimit override
	GasLimit *Uint64Quantity `json:"gasLimit,omitempty"`
	// EIP-1559 parameters, to be specified only post-Holocene
	EIP1559Params *Bytes8 `json:"eip1559Params,omitempty"`
}
```

基本的以太坊block 字段，加拓展的Transactions 列表等。这些组件最终会被op-geth 的execution engine 打包。

拓展阅读: [payload Attributes](https://specs.optimism.io/glossary.html?highlight=payload#payload-attributes)

# # 拓展
官网：[Derivation pipeline](https://docs.optimism.io/stack/rollup/derivation-pipeline)

