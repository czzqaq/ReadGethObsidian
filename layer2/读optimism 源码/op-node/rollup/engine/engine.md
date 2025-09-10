# engine Overview
 
```ccard
type: folder_brief_live
```
 

# 阅读顺序

- [Engine API](https://specs.optimism.io/protocol/overview.html?#engine-api) 理解engine 的职责是什么
- [[engine_controller.go]] 这里是L2 和 engine 交互的接口。
- [[events (build 流水线)]] 明白从engine 的动作到 L2 的过程。
- [[api.go]] 提供了 Execution layer 控制 block 的接口。

# 概念
## engine 
它是rollup 功能的入口。提供了一组 API。注意，这些 API 的调用不在 Optimism 的库里了，而是被外面的Execution 客户端调用，比如 op-geth。
指的是L2 engine，处理L2 block 的。包括sequencer 和 validator。

## unsafe block等
先理解 [safe](https://specs.optimism.io/interop/verifier.html?#safety) 的概念，然后从 [safe L2 block](https://specs.optimism.io/glossary.html?highlight=unsafe#safe-l2-block) 开始阅读。
safe 的根本含义是通过共识，在L2 的认知下，指的是在L1 上也有。

例如，unsafe L2 block 指没有同步到L1 的块。

