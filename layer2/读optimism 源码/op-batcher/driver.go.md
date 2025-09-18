
# 内容
就是启动batcher 服务。主要是开启3个channel 和 4个 loop。

## 四个loop

### blockLoadingLoop

```
每 PollInterval:
    syncStatus = getSyncStatus() // sequencer 的status
    blocksToLoad = syncAndPrune(syncStatus) 

    若 blocksToLoad != nil:
        loadBlocksIntoState(start,end)
        sendToThrottlingLoop(pendingBytes)
    trySignal(publishSignal)
```

其中:

`syncAndPrune` “prunes blocks and channels from its internal state which are no longer required”。 详见：[[sync_actions.go]] 获得具体哪些需要被prune，而从channel 中删除的意思见 [[channel_manager.go]]

`loadBlocksIntoState`: 它的block 来源是L2 client(客户端)，不在本optimism 的项目中实现。block 写入到[[channel_manager.go]] 

### publishingLoop

```
for range publishSignal:
    if !checkTxpool() { continue }     // 取消逻辑
    publishStateToL1()                 // 循环直到没有更多 txData
```

`publishStateToL1`：

1. 查询 `l1Tip()`。  
2. 调用 `channelMgr.TxData(...)`  
   - 返回 `io.EOF` = 没有数据。  
   - 返回 `txData{ frames, asBlob }`。  
3. 调用 `sendTransaction`。

如果启用了 `UseAltDA`：

- `publishToAltDAAndL1`  
  – 在受限 goroutine 中将 call-data 发布到 AltDA，  
  – 成功后构建 calldata 类型的承诺交易。  
  – 失败则通过 `recordFailedDARequest` 重新入队。

否则：

- 构建 **blob** 或 **calldata** 类型的交易候选。

所有候选交易都通过 `txmgr.Queue` 的 `queue.Send` 发送。


## 3个 channel

不同的loop 之间用channel 通信，例如，当 blockLoadingLoop 检测到 sequencer 的状态合适后，读取了block，发送给 Publish Loop 信号，来说明：“来活儿了”。它比较类似于 emit 和 on event 的驱动，平时，这些loop 的go routine 就单纯loop 着。

