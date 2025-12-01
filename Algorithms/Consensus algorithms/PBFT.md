# 是什么

## 背景
Practical Byzantine Fault Tolerance。原论文：[reference](http://pmg.csail.mit.edu/papers/osdi99.pdf)，知乎里一个不错的文章，[共识算法系列：PBFT算法关键点综述、优缺点总结](https://zhuanlan.zhihu.com/p/53897982)

关于 liveliness 和 safety，见 [[相关知识#safe 和 liveness]]

该算法的practical，其实就是给了拜占庭算法的一个实现。一个优化是，它的网络依赖 synchrony 提供liveness(因为在完全 asynchronous 的系统中实现 liveness+safety 不可能)，PBFT 假设网络不会完全异步，即等待有限的时间，所有消息都能正确送达。如果没有达到这种理想的 synchrony 假设，依然可以保证 safety，但是没有liveness，系统会卡住，永远无法进行下去。拓展阅读：[[相关知识#FLP 不可能原理]]

## faulty 容错

设总共有 **n 个节点**（relicas），其中：

- 最多允许有 **f 个节点**出现拜占庭错误（即作恶、宕机、发送错误信息等）。
- 这个 f 满足： 
$$f = \left\lfloor \frac{n - 1}{3} \right\rfloor
 $$

即严格少于 1/3. 

举个例子，如果系统可以允许 f=1，则至少有4个节点。


### 为什么 n>=3f+1
这个算是基础理论了，在很多教程里都提到了，具体怎么回事可以看论文[Asynchronous Consensus and Broadcast Protocols](https://dl.acm.org/doi/pdf/10.1145/4221.214134)

一个节点必须在跟 n-f 个副本通信后才能继续推进协议。因为 f 个有问题，你不能期待它们给你回应。
得到的 n-f 个响应中，最多有 f 是坏的，因为其实可能那 f 个有问题的都是网络故障才有问题，或者单纯相应慢被忽略了。

在这种极限情况下，回复里的诚实回复数量必须大于faulty 的，才能得到正确共识：
(n-f) - f > f，即 n > 3f

## 目的
m 是客户端发来的一个操作请求，比如“给我账户余额”，在replica 上全都运行一遍。如果每个 replica 都是诚实的，那么这个系统将会一致。
这个算法保证：
每个replica节点，按照相同的顺序，执行相同的请求。

我们把每次请求定为一个epoch，每次只有一个 epoch 在执行，保证了相同顺序，之后就由下面的验证行为确定执行的是相同的请求。

# 具体行为

## 概念
### 客户端
向主节点发送请求系统应答的 request，并追踪全部节点的应答。最终决定一致的consensus结果。

### 主节点
pre-prepare 阶段负责广播来自 客户端 的request。其他阶段等于一个普通节点。

### view
主节点由 view 号决定，p = v mod n。

## 流程
![[Pasted image 20250925152120.png]]
- **request**: client 给主节点发请求。
- **预准备（pre-prepare）**：主节点为请求分配序列号n，多播PRE-PREPARE消息（携带请求摘要）给备份，并将消息记入日志。
- **准备（prepare）**：备份收到并验证PRE-PREPARE消息后，发PREPARE消息给所有副本，并将消息记入日志。
- **提交（commit）**：副本（含主节点）收到足够（2f+1）PREPARE消息后，多播COMMIT消息。
- **reply**: 收到2f+1个匹配COMMIT消息后，副本运算操作m，提交包括了 执行 m 的结果在内的消息。
- **确认信息**：client  收到f+1 个相同回复，即可确认。

一般说流程是3步，pre-prepare~commit


### 为什么 2f+1
因为假设收到了 f 个恶意的节点的回应，那么起码多数的 f+1 还是诚实的。

### 为什么 f+1
client  收到 **f+1** 个相同回复，即可确认。
这是因为最多有 f 个节点捣乱，至少有一个诚实的节点在prepare 阶段确定了结果的真实性。

### 为什么还要有 commit 的确认
commit 指的是通知其他节点，我认同了当前节点下的concensus 是正确的。而在 commit 之后，每个节点还要再收集 2f+1 的prepared 消息。
这一步是为了保证，在 view change （也就是开始了一轮新流程） 的时候，当前的共识不可回滚。比如如果有两个 view 同时进行，实际上攻击者可以同时伪造两个 view 的内容，等于是一个 f 当两个用。
再比如下面的双花攻击的例子，先理解 [[#view change 流程]]。

#### 系统参数

- 副本总数 n = 3f + 1 = 7
- 最多坏副本 f = 2
- 达到 prepared 需要 2f + 1 = 5 份 PREPARE  
    （含主节点的 PRE-PREPARE）

设坏副本为 B₀、B₁，正确副本为 G₂…G₆。

#### 步骤 1：view 0 内把序号 n 分配给请求 a

1. 主 B₀ 发送 PRE-PREPARE(n,a) 给  
    B₀、B₁、G₂、G3、G4 这 5 台。
2. 这 5 台各自发送 PREPARE(n,a)。  
    ─► 每台都收齐 5 份准备消息 → 进入 _prepared_。
3.  B₀、B₁、G₂、G₃、G₄ 都立即向客户端回包 a。  
    客户端拿到足够多份的正确 reply，  
    于是确认操作 a 成功。

此时还有两台正确副本 G₅、G₆ 并不知道 a 的存在。


#### 步骤 2：换主，丢失 a

4. B₀ 这时失联，系统触发 view change，  
    新主选到了 G₂（正确）。
5. NEW-VIEW 需要收集 2f+1=5 份 VIEW-CHANGE。  
    可能出现的 5 份报告来自  
    G₂、G₅、G₆（3 个正确）＋ B₀(掉线不提交)、B₁（坏）。  
    其中只有 G₂ 报告了 _(n,a)_ 已 prepared；  
    另一台知道 a 的 G₃ 或 G₄ 可能没被选进这 5 份。
6. NEW-VIEW 规则：  
    若某序号在所收集的 VIEW-CHANGE 里 _没有_ 获得  
    “2f+1=5 份相同 prepared 证明”，  
    就用 _null_ 填这个序号。  
    结果 _(n,a)_ 不在新视图日志里 → 被“蒸发”。

### 为什么 primary 是恶意节点也没事
因为其他人报了 prepared，说明善意节点内就达成了一致。如果这个primary 说谎了，最多只有 f 个恶意节点赞同它。


## view change 流程
### 概念对齐

- view ：一轮普通的消息传递。
- view change：因为 primary node 失败，在足够长的时间里都没有得到 f+1 的commitment，为了让系统有有限的 liveness，还能继续跑下去，换一个新的 primary，继续运行。
- primary 即某一个特定view 里负责广播pre prepare 消息的，它实际上也决定了 view 的epoch。
- backup： 非primary 的replica
- checkpoint：每个消息 m，被执行后，产生的state 就是一个checkpoint。
- stable checkpoint：已经被证明的checkpoint。

### 论文中出现的符号

- v: view 编号
- i: 代表某一个backup
- σ(i）: i 的数字签名
- n: 最近一次 checkpoint 对应的n。
- **P** : P_m 的集合
- P_m: 一些数据的集合，表示对消息m 在n 之后的消息相应。它包含一个 PRE-PREPARE消息，2f 个 PREPARE 消息。由P_m 证明消息 m 本应该被 check in（但是没有，因为checkpoint 在之前）
- **C** : checkpoint message 的集合，为了理解checkpoint，见 4.2 garbage collection，简单来说，就是定期发一个对当前状态的验证消息，看是否有足够多的人同意这个状态。这些同意的 message 构成的集合。这个消息是用来证明消息中 n  的合法性，它确实是最新一次checkpoint 的n。
- **V**: 全部primary 收到的 VIEW-CHANGE,和它将要发出的VIEW CHANGE 消息。
- **O**：新view 的pre-prepare 消息。

### 过程

1. 由任意一个 backup 的超时触发。一个请求在时间内没有得到 commit，就是超时。这个备份称为 i，此时的view 是 v，切换到 v+1。
2. i 停止运行，不再接受消息。除了和 view change 有关的 CHECKPOINT VIEW-CHANGE NEW-VIEW。
3. 向全部节点广播 VIEW-CHANGE 消息，包括n，**C**，**P**
4. 当 promary node 得到 2f 个 VIEW-CHANGE 消息时，它广播 NEW-VIEW 消息。包括 **V** 和 **O**，**O** 开始了新的一轮 view。**O**的计算如下。
5.  NEW-VIEW 消息中包含了全量的消息log，据此每个backup 验证 **O** 的正确性，然后以这个新 view 为准，继续normal 流程，开始 PREPARE 的消息相应。

#### 计算 O
O 是在 NEW-VIEW 消息 中的，由primary计算。

计算 
*min-s*  = max{n_i}
max-s = max{ 所有 P 中的 PREPARE 消息中出现的序列号 }

对于 min-s ~ max-s 之间的每个 序号n'：
选择一个Pm，其 PRE-PREPARE message 的序列化n=n' 。
如果由多个，选n 最大的一个，如果一个都没有，构造空请求。

依据这个选择的PRE-PREPARE ，构造 **O**=PRE-PREPARE(v+1, n, d)，即v+1, 其他不变。


# 改进算法
tendermint, Hotstuff

#todo