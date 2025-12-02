
# 概念

## op-node


## eventLoop
见driver/state.go
每个node 都有这样一个loop。

# 阅读顺序

先从[[rollup]] 开始，它提供了从L1 到L2 数据的转换，以及和 EL 层沟通的API。

这个op-node 文件夹里有很多细功能，不过这里的 node 和 p2p 文件夹就是其中两个比较大的了。