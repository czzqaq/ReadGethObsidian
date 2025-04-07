#概念 

本章内容来自于：[eth2book by Ben Edgington ](https://eth2book.info/capella/part2/consensus/). beacon 中的源码并没有这么复杂，它是我学习的笔记整理。

# 共识

## 概念和澄清

共识是指多个借点或者代理在区块链上达成一致的能力，保证区块链不会分叉。我们常说的共识有POS，POW，POA 等等几种，但其实，比如POW 中的打包者选择确实是通过work，但整个共识系统里不止这些。比如说POS 共识，指的是通过slash 罚款的方式，达到“accountable safety”，也就是即使因为攻击，以太网出现了严重的分叉，也会让攻击者付出极大的代价。

其实共识是一个庞大的算法系统。正如下面的引用：

> This is a good point at which to mention that neither proof of work nor proof of stake is
>  a consensus protocol in itself. They are often (lazily) referred to as consensus protocols,
> but each is merely an enabler for consensus protocols.


## safe 和 liveness

safe 指的是任何节点的认知都是一致的，起码不会出现矛盾的最终状态。live 指的是链会不停运行。这是一对矛盾，因为很显然，liveness 要求的是在任何情况下，包括出错的时候，都继续运行的能力。

## eth 2.0 

过去的以太网上存在两条链，存放交易的数据链，另一个是beacon chain，它存放vote。以太网更新了若干次，包括合并beacon chain 和 shard chain，使用POS 共识等，进入了我们现在讨论的这个ETH 2.0。我们关心的共识算法，运行在 beacon 层。

beacon 层也是一条链，类似的，它也拥有绝对的 avalability， 每个人都可以查看链上的内容，比如[[#Attestation]] 

## POS 框架

### 成为validator

通过质押32ETH 成为validator。在每个epoch 开始时，会选择一部分成为validator，一部分validator 退出。为了网络的稳定，以及不让大量的恶意攻击者轻易进入validator，只有很少一部分的validator 可以被activate，和deactivate（申请退出）。[`CHURN_LIMIT_QUOTIENT`](https://eth2book.info/capella/part3/config/configuration/#validator-cycle) 描述了这个validator 排队activate 和 deactivate 的具体数字关系。一旦某个私钥储存ETH，它就进入activation queue，排队等待激活，激活后成为active validator。

一个私钥可以质押比如64ETH，被当做两个validator 看待。

### epoch 和 slot

每12个slot 一个epoch，slot 是一个小周期，长12秒，每个slot 都会有1/32 的 validator 成员进行投票(称为committee)，且预期会增长一个新区块，每个epoch 进行一次 attestation，包括justified 和 finalized block 的选择（Casper FFG），以及validator的签名等。


### Attestation
attestation 是beacon chain 上储存的主要内容，每个slot 都有一条单独的attestation。

```go
Attestation {
  aggregation_bits: Bitlist,      // 哪些验证者参与了这次投票（聚合后）
  data: AttestationData,          // 核心的投票内容
  signature: BLSSignature         // 聚合签名
}
```
其中，attestation data是：
```go
AttestationData {
  slot: uint64,                   // 投票发生的 slot（时刻）
  index: uint64,                  // 验证者所属委员会的索引 index
  beacon_block_root: Root,        // 被投票的区块的根（hash）
  source: Checkpoint,             // 上一个 justified checkpoint
  target: Checkpoint              // 当前的 epoch checkpoint（目标）
}
```

# LMD Ghost

## 命名

Latest Message Driven for LMD and Greedy Heaviest-Observed Sub-Tree for Ghost.

## 概念
这是一个链选择算法。用来解决分叉问题。出现分叉的原因是每个节点接受的消息可能不一样，于是它们对链的理解也不一样。于是，proposer 可能会基于错误的节点提出新block，最终让链不再形成一个链，这个状态会引起比如double spend 的问题。

这个算法用来让所有validator 对block chain 的head 形成共识。从而使得：1. block chain 确实会像是chain 一样增长，而不是变成树。 2. validator 对chain的形状的认知都一致。

## 算法
### 消息选择
所谓 Latest Message Driven——对于每个validator，选择它slot 编号最大的vote，作为唯一的有效vote。
具体的，只有满足下面条件的消息是有效的消息：

1. not too old
    每个validator 每个epoch 只会而且确实会投票一次，所以有效的vote 一定来自于这次epoch 或者上次epoch。太旧的投票会被丢弃，这是为了防止 decoy flip-flop attack，攻击者把较早的epoch 投票重放，从而用较少的stack 达成公告的权重。

2. not too new
    投票不能来自当前slot。因为划分到当前slot 的那1/32，应该是基于上一个slot 的结果做vote。

3. 不是slashable 的。
    存在[equivocation balancing attack](https://ethresear.ch/t/balancing-attack-lmd-edition/11853?u=benjaminion)，攻击者通过发送在同一个slot，但是投票给的更重的分支错误的vote，来让链分叉。target 不同的投票会在Casper FFG 中被slash，但是分叉选择的算法基于的是上一次justified 的block 中的记录，所以这些投票即使被slash 了还是可能存在。本质上还是Casper FFG 的slash 对LMD Ghost 没有直接影响，所以在这里增加对是否slashable 的检查。

4. 签名正确。

5. 被vote 的block root对应的block 是否已知。如果不知道，会尝试获取。


### 链选择

方法是贪心的：对于每个非叶子的节点（有分叉），都计算分叉上两个子树上的投票者的权重和，权重是stake（尽管初始是32ETH，但是有罚没的情况）。这样迭代，最终直到树的根为止，选择出一个链，作为子树，完成算法。
- 虽然定义时，这个链的统计必须从根开始，但配合 Casper FFG，从finalize block 开始即可。
- 所谓GHOST：Greedy Heaviest-Observed Sub-Tree.

### 围绕着stack 的奖惩
#### slash
所谓的 [nothing at stake problem](https://ethereum.stackexchange.com/questions/2402/what-exactly-is-the-nothing-at-stake-problem) 指，如果你的每个错误投票不被惩罚，那么你就会不加分辨的给每个选择都投票，所以重复的投票会被slash。重复的投票指，slot 相同的两个attestation data 中，存在不同的`beacon_block_root`

错误的head block 投票不会被slash。

#### incentive
正确的投票，加上被投票的block 成功被入链，validator 会被奖励。对于正确的打包，不会有专门的奖励，但是proposer 的block 被正确的加入到了标准链上本身就是很大的奖励。

显然，越靠后的validator，越容易做出正确的投票，因此，奖励是越靠前的validator 越多，详见 [inclusion delay](https://kb.beaconcha.in/ethereum-staking/attestation#rewards).

#### inactivity leak
(但是长时间的不参与投票会inactivity leak，这不是slash，因为validator不会失去资格，而且stack 不会直接清空，但也是stack 罚没，会让stack 逐渐减少。

## confirmation
在分支选择中被排除在链外的block 会被revert。block chain 上发生reorg （[[blockchain.go-chain head]]。所谓confirmation，就是确认了一个块几乎不可能被revert，因此可以放心的在上面部署合约，等等。

在 LMD Ghost 中，一个block $b$ 被confirm，当 
$$q_b^n > q_{min}$$
$q_b^n$ 指在 slot n 时，b 为根的子树上的总权重，占b 诞生以来全部投票权重和的比例。q_min 是一个超参数，是1/2 + β，β是认为被adversary 控制的stake 比例，是小于1/3的某个值。

另外，在hardhat 中，我见过这样的confirmation 确认方式：
 ```typescript
const tx = await contract.someFunction();
await tx.wait(5); // 等待 5 个区块确认
 ```

这里的确认方式是一种经验法则。两种confirmation 方法的目的都是相同的。



# Casper FFG
## Casper FFG 算法

### checkpoint 

在投票中，有字段source 和target。source 和target 都是checkpoint 格式：

```go
Checkpoint {
  epoch: uint64,                  // 纪元编号
  root: Root                     // 对应区块的根（hash）
}
```

epoch 是 POS 共识循环的概念，指的是在做出以 `root` 为根哈希的 block，在 `epoch` 的第一个slot 时，在proposer 看来，是区块的头。
 
### link
source->target，构成了一个 **link**。source 和 target 对应的block 一定在同一条链上，source 的epoch 一定小于target。
一般来说（也有特殊情况），一次投票的 target checkpoint，在下一次投票会变成 `source`
#### target
在 validator 看来，当前 `epoch` 的chain head 是`root`，就可以打包一个target了：
```
AttestationData.target = {
    epoch:epoch,root:root
    }
```

#### source
是 justified checkpoint。
1. source 一定选自上一个epoch 的某个 validator 的target 提议中。
2. 这个target 提议，是 [[#supermajority link]] 的target。

那么，就把这个checkpoint 拷贝给source：
$$source_{t} = target_{t-1},\ target \in superlink$$

### supermajority link
如果2/3的validator都给这同一个link 投票，那么这个link 就是一个 **supermajority link**。

#### justification checkpoint

supermajority 的target 是justified checkpoint.

#### finalisation checkpoint

1. supermajority 的source
2. target 是source 的直接子节点
3. source 已经被justified

满足以上3个条件的被称为finilisation checkpoint。

实际操作中，可以证明，如果存在c1->c3的supermajority link，c3不是c1的直接子节点，但是c2 是c1 的直接子节点，c3 是c2 的直接子节点，且c2 也被justified，那么也可以认为c1 是finalised。


### 更方便理解的一个例子
  

并不禁止target 不是 source 的直接子节点，尽管运转顺利的时候是。(但是，target 和 source确实必须在同一条chain 上)

甚至也不禁止source 不是上一次的target，所以[这样的投票](https://eth2book.info/capella/part2/consensus/casper_ffg/#exercise)是有可能的。
![[Pasted image 20250405231611.png]]

上面的例子表明，source的投票准则是：最新的 justified checkpoint，最新的定义和LMD Ghost 一样, 指vote 数据中的最大slot 投票，而不是checkpoint的height 最高，也不是网络层面中最近的消息。

## slash
### 什么时候 slash

下面两个规定被称为commandments，违反罚没stake，并逐出validator 队列。

1. **No double vote** 一个validator 不能给同一个高度的target 投两次票。注意是高度相同就不行。

2. **No surround vote** 两个投票 (S1→T1) 和 (S2→T2)，且满足 S2 < S1 < T1 < T2，那么这个投票就是surround vote。
![[Pasted image 20250405232002.png]]
### 补充
#### slash 的扣费情况
不一定会被罚没所有，如果**36天**的窗口内，只有少量的validator 违反了commandments，那么只会被罚没少量，如果违规达到1/3，则罚金达到最大，全部stake，这类似于在紧急情况下的自毁机制，因为在被攻击的时候才会出现这种情况。

但是注意，无论是不是被全部扣掉stack，validator 资格一定没了。

#### slash 保护

少数情况下诚实的validator 也有被罚没的风险，比如突然有一个高度很高的分支被revert了，这时候投票就会是surround vote。另外，把一个validator 私钥部署在两个物理节点，也很可能出问题。因此，绝大多数的客户端都会有slashing 保护。


## incentive 和惩罚
预测source 非常容易，因为source 是对justified 节点的说明，你除非完全工作在一个错误的环境里，否则很难看到的2/3跟别人不一样，因此target 的奖励更多。而预测错source 的vote 就不会得到奖励，反而会被惩罚。

完全不投票，和LMD Ghost 一样，会有少量惩罚（不是slash）。实际上不投票和投错误票的效果是一样的，占了2/3的分母，就得做出贡献才行，否则等于是阻碍了共识的达成。不及时的投票等于不投票，投票的入块时间和checkpoint 的slot之间必须间隔32个slot 以下。

此外，对于不投票的惩罚，还有 inactivity leak，这个情况下，有大量节点长期不投票，使得链在4个epoch 中都没有达到fininality，那么启动 inactivity leak 模式，每个epoch 都会大量惩罚未投票的validator，直到在线的validator 的stake 占比达到2/3，finaility 恢复。

## 作用

casper FFG 的作用是找到一个稳定的block。在 [[blockchain.go-chain head]] 中，就提到了finalize block 的作用，它是 **safe** 的，不可能被reorg（除非 >2/3 显著作恶），意味着你可以把它的状态作为一个可以依靠的检查点，并且从这里开始做Transaction execution、block 验证 和world state 更新。

casper ffg 提供了一种确定性的安全。
