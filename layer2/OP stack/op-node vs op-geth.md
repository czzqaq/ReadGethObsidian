op-node 相当于共识层，即我正在阅读的Optimism 源码。
op-geth 则相当于执行层，它和geth  的功能很像，但是beacon 部分依赖的改为了 op-node 的 Rollup engine。

见： [Engine API](https://specs.optimism.io/protocol/overview.html?#engine-api) 它提供了共识层的Optimism 给geth 暴露的部分。

