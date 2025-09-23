
# 说明

我不准备细看这部分代码，因为它就是一个实验性的feature。阅读 [# Alt-DA Mode Explainer](https://docs.optimism.io/stack/beta-features/alt-da-mode)，明白这部分功能做了什么。正常 DA 层都在 L1 上，而 Alt-DA 让 OP stack 使用其他 DA 层（如 Celestia、EigenLayer 等）来存储 L2 数据。


off chain 的DA 如何做 “availability”（可见性） 呢？提供了一个 challenge contract on L1。“ Challenging an input commitment means forcing providers to submit the input data on chain to prove the data is available during a time period.” 如果能在challenge的窗口内，把数据提交了，就通过，否则，把 derivation reset，即把提交到 DA 层的数据撤回。
关于这个 contract，见[[bindings-dataavailabilitychallenge]]

## 提交流程

这部分内容见 [specs](https://specs.optimism.io/experimental/alt-da.html#data-availability-challenge-contract), ALT DA 层的输入，肯定不再走 L1 service，而是先把batch 储存起来（DA Strorage layer）。

之后，有commit，这是把 block 序列化，并送入 DA server 的过程。这步会对输入做一些例如长度上的检查。不同的 DA server 有不同的序列化格式。

DA server 的交互对象有 challenge contract 的使用者、还有 derivation 的部分，毕竟它就等于是 DA Layer 了嘛。在 derivation 的部分，就有做challenge 。
