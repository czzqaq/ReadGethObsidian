
一个纯数据 struct。被 [[status]] 更新。

它用于描述节点当前的同步（Sync） status。包含

- L1 状态
    - headL1：
    - SafeL1：
    - Finalized L1
    - Current L1 在 derivation 中正在处理的L1 区块。
关于几个block 的理解，详细阅读 [[POS 共识]]

- L2 状态
    - UnsafeL2：指 sequencer 刚刚生成的最新区块
    - safeL2，从L1 derivation 过来的区块。
    - FinalizedL2：对应L1 中的finalize 区块
    - Interop 字段。和 [[superchain.go]] 有关，跨链操作时，产生的本地safe 和跨链safe 的区别。
关于L2 上safe，见 [[engine#unsafe block等]]




