
# 概念

对L2 的链做管理的角色。在L1 里这部分是由矿工的共识完成的

链做管理相关功能：
- 选择交易的顺序与打包；
- 选择对应的 L1 起源（L1 origin）并设定区块时间戳等属性；
- 构建、封印、传播 L2 区块，并把结果提交给执行引擎与外部网络；
- 在与 L1 同步或出现错误时进行退避、重试或重置；
- 硬分叉、策略调整。

---

进入L2 的数据有：
- 用户直接的L2交易
- L1 观察到的deposit(derivation 提供的attributes)
- 一般的(regular) 交易

sequencing 的过程是把regular 交易打包到L2 链的过程。


# 阅读顺序
- 应该先完成[[derive]] 的阅读。
- 阅读官方的定义 [sequencer](https://specs.optimism.io/background.html?highlight=sequenc#sequencers)
- 从 [[sequencer.go]] 开始阅读源码

