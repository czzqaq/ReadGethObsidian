# 概念

我们知道，batcher 的作用是把L2 的区块，提交到L1. block 被打包为channel。
derive 的动作中也存在 [[frame and channel]] 的概念，它也是block 的集合，不过是 L2 的。把 block data 组成帧的过程，和 derive 共享一处代码。

## channelBuilder
提供了产生 channel 的功能，主要是对帧的产生和管理。

输入是从 AddBlock 开始的，输入L2 block，然后L2 block 被序列化，切分，构成 帧。
所谓channel Builder，其实就是一个 frame queue.提供了NextFrame() 功能。

## 交易确认
channel 主要多出来的事，包括失败，超时等情况的处理。如果交易确认，则channel.close() 。

