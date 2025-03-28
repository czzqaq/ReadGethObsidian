# 作用

`triePrefetcher` 是一个性能优化组件，作用是：

> **在状态访问发生前，预测并加载相关的 trie 节点到缓存中，以加速后续访问并减少数据库读取开销。**

## 使用例

1. 在 [[blockchain.go#insertChain]] 中，`statedb.StartPrefetcher`，以当前的 worldstate 的Root Hash，和 world state 的trie database ，开启一个Trieprefetcher. 
2. 调用 s.prefetcher.prefetch 方法，启动了一个 subprefetcher goRoutine。并且 `schedule` 了一批应该被写入的状态（比如一个交易涉及到哪些 account，就把这些account 在 `prefetch` 中传入到schedule ，与其相关的状态都会被添加。 
3. 在 `subfetcher.loop` 中，不断往绑定的 trie 内存中写入状态。