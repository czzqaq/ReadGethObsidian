# 概括说明

提供了：
- OpenBlock 依据上一个parent block，以及新的交易 attributes，开始对链生长。
- CancelBlock 用于清理一个build 的记录
- SealBlock 获取构建好的区块，这是个只读函数
- CommitBlock 输入一个具体的完成了的block，推进链的状态

这几个函数的实现中，CancelBlock，SealBlock 都非常简单，而 CommitBlock 也就是 engine_controller 的简化版，OpenBlock 的实现不在我现在的程序里，通过 RPC 去一个别的什么地方运行 engine_forkchoiceUpdatedVx 了。
详见背景。


# 背景
注意这套 API 是对 L2 上的 以太坊 engine API 的包装。在L1上，实际上也有类似 `engine_forkchoiceUpdatedV1` 的API，见：
来自登链的：[以太坊 Engine API：可视化执行层和共识层之间通信流程](https://learnblockchain.cn/article/19306)
内容很类似的英文版 [Engine API: A Visual Guide](https://hackmd.io/@danielrachi/engine_api)
来自 Optimism 的：[Engine-api-specs](https://specs.optimism.io/protocol/exec-engine.html?highlight=engine_getPayloadV2#engine-api)
以太坊官方的各种markdown：[github](https://github.com/ethereum/execution-apis/tree/main/src/engine)



## engine_forkchoiceUpdated
 CL 调用 EL 的方法，共识层用于通知执行层链状态更新的标准接口。调用场景比如 safe block 更新，以及共识层通知执行层构建新区块。


### payload
在通知执行层区块的内容时，使用了：


 ```go
 PayloadAttributesV2: {
    timestamp: QUANTITY
    prevRandao: DATA (32 bytes)
    suggestedFeeRecipient: DATA (20 bytes)
    withdrawals: array of WithdrawalV1
    transactions: array of DATA
    noTxPool: bool
    gasLimit: QUANTITY or null
}
 ```

可以注意到，这里提供的 prevRandao，以及timestamp。这是单纯给 layer 2 的执行层用的，内容有精简。


### block building 过程

![[Pasted image 20250913192949.png]]

这个消息从 concensus layer 到 Execution Layer，触发是从一个block 被propose 开始。

engine_forkchoiceUpdated 带 payload 给 EL。EL 回成功。

然后第二步的get payload 实际上就是 SealBlock 的过程，它获得带 state root 等执行后的block 的内容。