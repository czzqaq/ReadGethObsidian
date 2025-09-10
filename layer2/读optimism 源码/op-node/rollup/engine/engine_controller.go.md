提供对链的情况的记录，即safe head 等状态的更新。确保 L2 状态和L2 engine 中保持一致。
关于 safe 的概念，见 [[engine#unsafe block等]]

核心是 EL（Execution Layer） sync 功能: 

```go
const (
    syncStatusCL syncStatusEnum = iota
    // We transition between the 4 EL states linearly. We spend the majority of the time in the second & fourth.
    // We only want to EL sync if there is no finalized block & once we finish EL sync we need to mark the last block
    // as finalized so we can switch to consolidation
    // execution client is running in archive mode. In some cases we may want to switch back from CL to EL sync, but that is complicated.
    syncStatusWillStartEL               // First if we are directed to EL sync, check that nothing has been finalized yet
    syncStatusStartedEL                 // Perform our EL sync
    syncStatusFinishedELButNotFinalized // EL sync is done, but we need to mark the final sync block as finalized
    syncStatusFinishedEL                // EL sync is done & we should be performing consolidation
)
```

 `InsertUnsafePayload` 是最核心的函数，它向 Execution Engine 发送 `engine_newPayloadV1` 请求，插入新区块。
