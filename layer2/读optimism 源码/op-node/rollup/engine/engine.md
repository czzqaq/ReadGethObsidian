# engine Overview
 
```ccard
type: folder_brief_live
```
 

# 阅读顺序

- [Engine API](https://specs.optimism.io/protocol/overview.html?#engine-api) 理解engine 的职责是什么
- [[engine_controller.go]] 这里是Rollup 功能的调用入口
- [[api.go]] 提供了 Execution layer 控制 block 的接口。

# 概念
## engine 
它是rollup 功能的入口。提供了一组 API。注意，这些 API 的调用不在 Optimism 的库里了，而是被外面的Execution 客户端调用，比如 op-geth。
