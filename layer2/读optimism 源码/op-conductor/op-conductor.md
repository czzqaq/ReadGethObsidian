#todo 

# 概念

## 作用
1. 这是一个辅助的**服务** service
2. 和 Sequencer 并行工作，阅读 [sequencer](https://specs.optimism.io/background.html?highlight=sequencer#sequencers)
3. 表现得仿佛是一个 concensus layer，决定 sequencer 的“leader”（？），unsafe block ，检查 sequencer 的health（在线等等）
4. 但是注意，它并不跑共识算法，所有的sequencer 这里都是可信的。

## sequencer 和 conductor
![[Pasted image 20250925081055.png]]


总之我要明白 sequencer 是什么，他们是产生 L2 的用户没错，整个L2 上的流程它能解决掉，但是不安全，于是需要 batcher 提供 DA，以及可能有后续的derivation 来解决L1 和 L2 之间的一致性问题。[[sequencing]]

这里的leader 就是分布式数据库里的概念，见 [raft](https://zhuanlan.zhihu.com/p/643338097). batcher 最终只依赖 leader 的结果，而 conductor 负责选出leader，和保证各个sequencer 与leader 同步。
这让人不免疑惑，sequencer 的集群有什么好处吗？结果研究了一下，好像它主要就起一个灾备效果。

## 使用例
假设我有3个 sequencer，下面是一个不保证绝对可用的例子，单纯是列出了完整的过程。

在每个 sequencer 上运行：
```bash
export OP_CONDUCTOR_CONSENSUS_ADVERTISED=<本机IP:50050>
export OP_CONDUCTOR_RAFT_SERVER_ID=<唯一ID>   # 0/1/2
export OP_CONDUCTOR_NODE_RPC=http://localhost:8545
export OP_CONDUCTOR_EXECUTION_RPC=http://localhost:8546
docker run ... op-conductor:latest

OP_NODE_CONDUCTOR_ENABLED=true 
OP_NODE_RPC_ADMIN_STATE="" # 必须为空
```

启动一个conductor 容器。并开启conductor 功能。



对 sequencer 0 和 2（未来做follower的）暂停
```bash
curl -X POST .../conductor_pause
```

将 sequencer 1 设置为集群的创建开始者，即做conductor 。
```bash
OP_CONDUCTOR_RAFT_BOOTSTRAP=true
```

这样，1号sequencer 成为 leader。可以观察到它持续产出 unsafe block。

把 0/2 加入集群
```bash
# 在 1 号 conductor 上
curl .../conductor_addServerAsVoter 0 <0的IP:50050> 1
curl .../conductor_addServerAsVoter 2 <2的IP:50050> 1
```

恢复sequencer 运行
```bash
curl .../conductor_resume     # 分别对 0/1/2 执行
```

故障时，切换
```bash
# 主动把领导权转给 0 号
curl .../conductor_transferLeaderToServer 0 <0的IP:50050> 1
```

# 阅读代码
这部分分了很多文件夹，不过其实基本上每个文件夹里都只有一两个文件。它们分别是：

- consensus 
    - 核心逻辑，以 Raft 算法为核心，有一个状态机。见 [[concensus-raft]]
- conductor
    - 服务入口，主要是服务启停，unsafehead 追踪等。见 [[conductor-service]]
- client
    - sequencer 接口的包装，用来提供对sequencer的控制。比较简单。
- Health
    - 和区块链业务基本无关，通过p2p 来检查。见 [[health-monitor]]
- 其他
    - metrics，rpc 功能等。

