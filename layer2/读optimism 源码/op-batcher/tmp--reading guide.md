gpt 生成

# Optimism **BatchSubmitter / driver.go** – 阅读指南

---

## 0. 为什么需要 driver？

**BatchSubmitter** 是将新生成的 L2 区块传输到所选数据可用性（DA）介质（Ethereum L1 calldata、blobs 或 Alt-DA 服务）的桥梁。  
因此它：

1. 跟踪 sequencer 对链的视图，  
2. 将区块处理为 “channels → frames → transactions”，  
3. 发送这些交易，  
4. 观察交易回执，  
5. （可选）当 DA 积压过多时限制 sequencer 的速率。

如果它停止工作，L2 数据最终将不可用，Rollup 将停止运行。

---

## 1. 建议的阅读顺序

| 步骤 | 文件 / 符号 | 为什么从这里开始？ |
|------|---------------|---------------------|
| 1 | `BatcherConfig`（同包） | 了解后续使用的所有配置项。 |
| 2 | `NewBatchSubmitter` 和字段 | 了解重要的协作者（TxMgr、ChannelManager、EndpointProvider…）。 |
| 3 | `StartBatchSubmitting` | 入口点；启动所有 goroutine，连接 channel。 |
| 4 | `blockLoadingLoop` | 将新 L2 区块输入整个流水线。 |
| 5 | `publishingLoop` | 消费区块，创建 channel/frame 并生成交易队列。 |
| 6 | `sendTransaction` 及辅助函数（`blobTxCandidate`, `calldataTxCandidate`） | 了解原始字节如何变成 L1/DA 交易。 |
| 7 | `receiptsLoop` | 回执如何影响内部状态与指标。 |
| 8 | `throttlingLoop` / `singleEndpointThrottler` | 保护 DA 可用性的流控逻辑。 |
| 9 | `channelManager` 包 | 实际缓存区块和 frame 的数据结构。 |
| 10 | `txmgr` 包（op-service） | driver 复用的通用 nonce/GasPrice 管理器。 |

## 2. 需要理解的核心依赖

1. **channelManager**（op-batcher）  
   - 包含 `blocks` 映射和 `channelQueue`。  
   - 将 *unsafe blocks → frames → txData*。  
   - 提供接口：`AddL2Block`, `TxData`, `TxConfirmed`, `TxFailed`, `Clear`, `Prune*`, `PendingDABytes()`。

2. **txmgr.TxManager / txmgr.Queue**（op-service/txmgr）  
   - 提供异步 `Send`、`Wait`、`IsClosed`，处理 nonce 与 tip-cap 管理。  
   - 返回 `txmgr.TxReceipt[T]` 到调用方提供的 channel。

3. **derive.L2BlockToBlockRef**  
   - 工具函数，用于将 `types.Block` 转换为 Optimism 内部的 `BlockRef`。

4. **EndpointProvider**（dial.L2EndpointProvider）  
   - 简单封装，允许 driver 获取 `EthClient` 和 `RollupClient` 句柄（适用于两个 JSON-RPC 命名空间）。

5. **altda.DAClient**（可选）  
   - 类 RPC 的客户端，将原始 channel 数据存储在链下，并返回小型承诺，从而降低 calldata 成本。

---

## 3. 生命周期概览

### 3.1 启动阶段

1. `StartBatchSubmitting`  
   - 防止重复启动，设置 `running=true`。  
   - 创建两个生命周期上下文：  
     – `shutdownCtx`（正常停止），  
     – `killCtx`（外部 ctx 被触发时强制停止）。  
   - 等待 **L2 创世时间** + （可选）节点追赶。  
   - 初始化状态（`txpoolState = TxpoolGood`）。  
   - 创建三个内部 channel：  
     – `pendingBytesUpdated` (`int64`) → throttling，  
     – `publishSignal` (`struct{}`) → publishing，  
     – `receiptsCh` → receipts loop。  
   - 启动 goroutine：  
     – `blockLoadingLoop`（始终启动），  
     – `publishingLoop` 和 `receiptsLoop`，  
     – `throttlingLoop`（如果 `ThrottleThreshold > 0`）。

### 3.2 blockLoadingLoop（生产者）

**正常路径**伪代码：

```
每个 PollInterval:
    syncStatus = getSyncStatus()
    blocksToLoad = syncAndPrune(syncStatus)

    if blocksToLoad != nil:
        loadBlocksIntoState(start,end)
        sendToThrottlingLoop(pendingBytesUpdated)
    trySignal(publishSignal)
```

关键辅助函数：

- `syncAndPrune`  
  – 使用 `computeSyncActions` 决定是否 GC 或完全重置。  
  – 更新 `prevCurrentL1`。

- `loadBlocksIntoState`  
  – 遍历区块号，通过 L2 RPC 获取并存入 ChannelManager。  
  – 检测重组 → 触发 `waitNodeSyncAndClearState`。

### 3.3 publishingLoop（消费者 / 分发者）

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

### 3.4 receiptsLoop

对每个 `TxReceipt[txRef]`：

- 检测 `txpool.ErrAlreadyReserved` → 将状态标记为 `TxpoolBlocked`，下次 `checkTxpool` 将发送相同类型（blob 或 calldata）的 **取消交易**。  
- 将记账工作委托给 `recordConfirmedTx` / `recordFailedTx`。

### 3.5 throttlingLoop

```
for pb := range pendingBytesUpdated:
    throttling = pb > ThrottleThreshold
    对每个 endpoint:
        尝试向 updateChan 发送 struct{}
```

每个 endpoint 对应一个 `singleEndpointThrottler`：

1. 调用 RPC 方法 `miner_setMaxDASize(maxTxSize,maxBlockSize)`。  
2. 失败时每 10 秒重试。  
3. 如果 RPC 方法缺失 → **停止整个 batcher**（硬性要求）。

当非 throttling 时，`maxTxSize` / `maxBlockSize` 设置为 0；当 backlog 存在时设置为较低值。

---

## 4. 将 README 步骤映射到实际代码

| README 步骤            | 代码入口                                                                    | 说明       |
| -------------------- | ----------------------------------------------------------------------- | -------- |
| “查询 syncStatus”      | `getSyncStatus` + `blockLoadingLoop` 中的调用                               | 状态为空时退避。 |
| “修剪 blocks/channels” | `syncAndPrune`, `channelMgr.Prune*`, `channelMgr.Clear`                 |          |
| “加载 unsafe blocks”   | `loadBlocksIntoState` → `loadBlockIntoState` → `channelMgr.AddL2Block`  |          |
| “加载后发出信号”            | `sendToThrottlingLoop`, `trySignal(publishSignal)`                      |          |
| “需要时入队新 channel”     | `channelMgr.TxData`（内部创建 channel）                                       |          |
| “压缩 & 分帧”            | `channelMgr.TxData`（调用内部 frame 构造器）                                     |          |
| “发送 frames 到 DA”     | `sendTransaction` 路径（AltDA 或 L1）                                        |          |
| “接收回执”               | `txmgr.Queue.Send` 启动的 goroutine 将结果写入 `receiptsCh`，由 `receiptsLoop` 消费 |          |
| “限制 sequencer”       | `throttlingLoop` + `singleEndpointThrottler`                            |          |
