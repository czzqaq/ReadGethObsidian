# computeSyncActions

主要就是这一个函数。

## 输入
```
newSyncStatus   Sequencer 对外暴露的 SyncStatus  (当前/安全/不安全头等)
prevCurrentL1   上一次看到的 Current L1 头
blocks          本地 blocks 队列
channels        本地待确认 Channel 列表
l               logger
```

### blocks

其中，blocks 指L2 block，在 [[driver.go#blockLoadingLoop]] 中被确定。这里的代码写的不明显，实际上computeSyncActions 的输出恰好就是它下一次的输入。它被称为 “state”，这里等于是状态机，unsafe L2 blocks 的block number 就是核心的状态，此外还有 L1 的情况：

### channel
channels —— 已打包但可能仍在 L1 等待确认的 Channel 列表

这些 channel 的申请在[[driver.go#publishingLoop]]，如果所有的 channel 都满了，就来个新的。anyway，channel 理解为L1 数据输入的地方就好了。
具体的channel 的概念见：见 [[channel.go]]


### newSyncStatus
newSyncStatus 包括下面这些
```go
// SyncStatus is a snapshot of the driver.
// Values may be zeroed if not yet initialized.
type SyncStatus struct {
	// CurrentL1 is the L1 block that the derivation process is last idled at.
	// This may not be fully derived into L2 data yet.
	// The safe L2 blocks were produced/included fully from the L1 chain up to _but excluding_ this L1 block.
	// If the node is synced, this matches the HeadL1, minus the verifier confirmation distance.
	CurrentL1 L1BlockRef `json:"current_l1"`
	// CurrentL1Finalized is a legacy sync-status attribute. This is deprecated.
	// A previous version of the L1 finalization-signal was updated only after the block was retrieved by number.
	// This attribute just matches FinalizedL1 now.
	CurrentL1Finalized L1BlockRef `json:"current_l1_finalized"`
	// HeadL1 is the perceived head of the L1 chain, no confirmation distance.
	// The head is not guaranteed to build on the other L1 sync status fields,
	// as the node may be in progress of resetting to adapt to a L1 reorg.
	HeadL1      L1BlockRef `json:"head_l1"`
	SafeL1      L1BlockRef `json:"safe_l1"`
	FinalizedL1 L1BlockRef `json:"finalized_l1"`
	// UnsafeL2 is the absolute tip of the L2 chain,
	// pointing to block data that has not been submitted to L1 yet.
	// The sequencer is building this, and verifiers may also be ahead of the
	// SafeL2 block if they sync blocks via p2p or other offchain sources.
	// This is considered to only be local-unsafe post-interop, see CrossUnsafe for cross-L2 guarantees.
	UnsafeL2 L2BlockRef `json:"unsafe_l2"`
	// SafeL2 points to the L2 block that was derived from the L1 chain.
	// This point may still reorg if the L1 chain reorgs.
	// This is considered to be cross-safe post-interop, see LocalSafe to ignore cross-L2 guarantees.
	SafeL2 L2BlockRef `json:"safe_l2"`
	// FinalizedL2 points to the L2 block that was derived fully from
	// finalized L1 information, thus irreversible.
	FinalizedL2 L2BlockRef `json:"finalized_l2"`
	// PendingSafeL2 points to the L2 block processed from the batch, but not consolidated to the safe block yet.
	PendingSafeL2 L2BlockRef `json:"pending_safe_l2"`
	// CrossUnsafeL2 is an unsafe L2 block, that has been verified to match cross-L2 dependencies.
	// Pre-interop every unsafe L2 block is also cross-unsafe.
	CrossUnsafeL2 L2BlockRef `json:"cross_unsafe_l2"`
	// LocalSafeL2 is an L2 block derived from L1, not yet verified to have valid cross-L2 dependencies.
	LocalSafeL2 L2BlockRef `json:"local_safe_l2"`
}

```

