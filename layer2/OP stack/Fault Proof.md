# 组件
## Pre-image Oracle

pre-image ，原像，指的是做交易之前的状态。Oracle 是一种把外部数据引入区块链的工具或者服务。Pre-image Oracle 引入的是VM 里的状态，目的地是客户端。也就是说，使用客户端的验证者，通过pre-image Oracle，来对VM 状态做query。

Oracle 导入的有两部分内容：

- The initial inputs to bootstrap the program. See [Bootstrapping](https://specs.optimism.io/fault-proof/index.html#bootstrapping).
- External data not already part of the program code. See [Pre-image hinting routes](https://specs.optimism.io/fault-proof/index.html#pre-image-hinting-routes).

其中Bootstrapping 部分的数据来自L1 VM。
Hinting 从链上取数据，是 JSON-RPC 。
## program

程序分为三个部分：
Prologue、Main Content、 Epilogue
上述的通过pre-image Oracle 导入数据的 Bootstrapping 过程发生在Prologue，加载输入过程。
Main, 处理L2 的状态转换，从L1 推导L2. 这部分和Rollup 的derivation 流程基本相同，区别仅在于：1. 状态变化保存在内存 2. 不通过RPC而是通过pre-image oracle 加载数据。
Epilogue, 验证状态的变化。得到是否通过验证的结论。

Program 全称 Fault Proof Program。运行在 FPVM 中。是执行state transition 的主体，等于 a combination of `op-node` and `op-geth`。

## VM
FPP 的包装，提供低级的指令、内存等等运行时环境吧。
- 一系列 Microprocessor without Interlocked Pipeline Stages，比如 load 到寄存器，加法，减法等。
- 内存管理 `mipsevm`


## dispute game protocol

作为最高级的抽象，先定义一个resolve 方法，其中claim 是一个H256。
```
resolve(claim) -> bool
```

一个state machine。包括：
IN_PROGRESS：并不知道claim 的真假。
CHALLENGER_WINS: claim 为假
DEFENDER_WINS: claim 为真。

具体的，有：FaultDisputeGame。可能有别的种类，但先就学这种了。
### Bisection Game
是OP stack 实现的第一个Dispute game
 有一个根claim。如果challengercounter 了它，则分叉继续寻找，直到找到细致的Dispute。
 显然有准则：
 - 定义trace index，指claim验证执行的步骤编号，则index 大的被counter，更早期的步骤就不需要再验证
 - 如果父claim 不诚实，子claim 也不诚实。

# 综述
## 原则
关键是去中心化、模块化。具体的：
- 任何人都可以提交关于 L2 状态的提案。任何人也都可以挑战提案。
- 从L2到L1 的消息和withdraw 均无需依赖第三方受信实体。
- 存在program、vm、dispute  game protocol 三个组件。每个组件都有多种实现（未来）

## 拓展阅读
[OP stack fault proof 系统解读](https://cloud.tencent.cn/developer/news/1203409 )
