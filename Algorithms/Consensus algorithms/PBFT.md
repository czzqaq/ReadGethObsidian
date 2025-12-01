# 是什么

## 背景
Practical Byzantine Fault Tolerance。原论文：[reference](http://pmg.csail.mit.edu/papers/osdi99.pdf)，知乎里一个不错的文章，[共识算法系列：PBFT算法关键点综述、优缺点总结](https://zhuanlan.zhihu.com/p/53897982)

关于 liveliness 和 safety，见 [[相关知识#safe 和 liveness]]

该算法的practical（P之于PBFT)，其实就是给了拜占庭算法的一个实现。一个优化是，它的网络依赖 synchrony 提供liveness(因为在完全 asynchronous 的系统中实现 liveness+safety 不可能)，PBFT 假设网络不会完全异步，即等待有限的时间，所有消息都能正确送达。如果没有达到这种理想的 synchrony 假设，依然可以保证 safety，但是没有liveness，系统会卡住，永远无法进行下去。实际情况下，靠着消息重传和节点的排除，可以达到弱liveness。
拓展阅读：[[相关知识#FLP 不可能原理]]

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

我们把每次请求定为一个epoch，由一个序列号n 标识。执行 epoch 必须按顺序，而commit 确实不用，因为commit 的含义是对prepare 消息真实性的承诺。

# 具体行为

## 概念
### 客户端
负责控制请求间隔。
向主节点发送请求系统应答的 request，并追踪全部节点的应答。最终决定一致的consensus结果。

### 主节点
pre-prepare 阶段负责广播来自 客户端 的request。其他阶段等于一个普通节点。
主节点的确定由view 的编号 v 确定。primary(v) = v mod n

## 流程
![[Pasted image 20250925152120.png]]
- **request**: client 给主节点发请求。
- **预准备（pre-prepare）**：主节点为请求分配序列号n，多播PRE-PREPARE消息（携带请求摘要）给备份，并将消息记入日志。
- **准备（prepare）**：备份收到并验证PRE-PREPARE消息后，发PREPARE消息给所有副本，并将消息记入日志。
- **提交（commit）**：副本（含主节点）收到足够（2f+1）PREPARE消息后，多播COMMIT消息。
- **reply**: 收到2f+1个匹配COMMIT消息后，副本运算操作m，提交包括了 执行 m 的结果在内的消息。
- **确认信息**：client  收到f+1 个相同回复，即可确认。

一般说流程是3步，pre-prepare~commit

### 状态

#### prepared(m, v, n, i)
在副本 i 看来，「请求 m 在视图 v 的序号 n 上的提议已经**被 2f+1 个副本认可为一致的候选**」，即本视图内的全序局部确定。

`prepared(m, v, n, i)` 为真，当且仅当副本 i 的日志里同时有：

1. 客户端请求 `m` 本体；
2. 一条 `PRE-PREPARE(v, n, D(m))`：
    - 来自 primary(v)；
    - 视图号 = v，序号 = n，digest = D(m)；
3. 至少 `2f` 条来自**不同备份节点**的 `PREPARE(v, n, D(m))` 消息，
    - 这些 PREPARE 的 `(v, n, digest)` 都和那条 PRE-PREPARE 匹配。

这里的2f+1 的+1 有自己确认自己。

#### committed(m, v, n)
全局来看，「请求 m 在视图 v 的序号 n 已经在**足够多的正确副本中**达成 prepared」，因此这个 (v,n) 最终一定只能对应 m，不会被别的请求占用。

`committed(m, v, n)` 为真：

存在一个**由至少 2f+1 个非拜占庭副本组成的集合 Q**，对每个 `i ∈ Q` 都有：

prepared(m,v,n,i)=true

#### committed-local(m, v, n, i)

从副本 i 的角度，「我已经确信 (v, n) 这个位置上的请求就是 m，未来不会被其他请求替代；因此我可以在本地把它当作‘最终决定’」。

在副本 i 上，`committed-local(m, v, n, i)` 为真，当且仅当：

1. `prepared(m, v, n, i)` 已经为真；**并且**
2. i 收到了来自**不同副本**的至少 `2f+1` 条 `COMMIT(v, n, D(m))` 消息：
    - 这些 COMMIT 与对应的 PRE-PREPARE 匹配，即 `(v, n, digest)` 一致。


#### 执行

副本 i **只有在**满足以下两个条件时才真正执行请求 m：

1. `committed-local(m, v, n, i)` 为真（说明 m 在 (v,n) 上已经“铁板钉钉”）；
2. i 的本地状态已经**按顺序执行完所有 `序号 < n` 的请求**。

### 为什么 2f+1
因为假设收到了 f 个恶意的节点的回应，那么起码多数的 f+1 还是诚实的，诚实节点一定大于恶意节点的数量。

### 为什么确认消息是 f+1
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


## view change 流程
### 概念对齐

- view ：一轮普通的消息传递。
- view change：因为 primary node 失败，在足够长的时间里都没有得到 f+1 的commitment，为了让系统有有限的 liveness，还能继续跑下去，换一个新的 primary，继续运行。
- primary 即某一个特定view 里负责广播pre prepare 消息的，它实际上也决定了 view 的epoch。
- backup： 非primary 的replica
- checkpoint：每执行到某个固定序号间隔（例如每 100 或 128 个请求），才生成一个 checkpoint 并进行 checkpoint 协议，形成对该状态的一致“证明”。
- stable checkpoint：已经被证明的checkpoint。

### 论文中出现的符号

- v: view 编号
- i: 代表某一个backup
- σ(i）: i 的数字签名
- n: 最近一次 checkpoint 对应的n。
- **P** : P_m 的集合
- d, digest，即消息的哈希。
- P_m: 一些数据的集合，表示对消息m 在n 之后的消息相应。它包含一个 PRE-PREPARE消息，2f 个 PREPARE 消息。由P_m 证明消息 m 本应该被 check in
- **C** : checkpoint message 的集合，为了理解checkpoint，见 4.2 garbage collection，简单来说，就是定期发一个对当前状态的验证消息，看是否有足够多的人同意这个状态。这些同意的 message 构成的集合。这个消息是用来证明消息中 n  的合法性，它确实是最新一次checkpoint 的n。
- **V**: 全部primary 收到的 VIEW-CHANGE,和它将要发出的VIEW CHANGE 消息。
- **O**：新view 的pre-prepare 消息。

### 过程（基于论文 §4.4）

1. **触发 view change**

   - 每个 backup 只要有“还没执行完”的请求，就启动一个超时计时器；
   - 如果等太久（例如一直收不到某请求的 PRE-PREPARE / COMMIT），计时器到期：
     - 它怀疑当前 view v 的 primary 出问题；
     - 决定切换到 view v+1（发起 view change）。

   此时该 backup：
   - 不再接受旧 view v 的 REQUEST / PRE-PREPARE / PREPARE / COMMIT；
   - 仍会接受 CHECKPOINT / VIEW-CHANGE / NEW-VIEW。

2. **备份发送 VIEW-CHANGE**

   触发 view change 的 backup i 构造并广播：

$$
   \text{VIEW-CHANGE}(v+1, n, C, P)_i,σ(i)
   $$

   其中：

   - `n`：i 所知的最新 **stable checkpoint** 的序列号；
   - `C`：该 checkpoint 的 2f+1 条 CHECKPOINT 消息（证明某个 digest 对应的状态在 2f+1 个副本上一致）；
   - `P`：所有 `seq > n` 且在 i 上已经 `prepared` 的请求的证明集合：
     - 每个元素包含：对应的 PRE-PREPARE 以及至少 2f 条匹配的 PREPARE。

3. **新 primary 收集 2f+1 个 VIEW-CHANGE，构造 NEW-VIEW**

   - 新视图 `v+1` 的 primary p 是 `p = (v+1) mod n`；
   - 当 p 收到来自不同副本至少 2f+1 条合法的 VIEW-CHANGE(v+1, n_i, C_i, P_i) 后：

     1. 形成集合 `V = {所有 VIEW-CHANGE}`；
     2. 计算：
        - `min-s`：所有 VIEW-CHANGE 中 stable checkpoint 序号的最大值；
        - `max-s`：所有 P_i 中出现的 prepared 请求的最大序列号；
     3. 对区间 `(min-s, max-s]` 内每个序号 s：
        - 若 V 中至少有一个关于 s 的 prepared 证明：
          - 选视图号最高的那个证明，对应的请求 digest 为 d\*；
          - 加入 PRE-PREPARE(v+1, s, d\*) 到集合 O；
        - 否则：
          - 把该 s 填成空操作：PRE-PREPARE(v+1, s, D(null))；
     4. primary p 构造并广播：

        $$
        \text{NEW-VIEW}(v+1, V, O)_p,σ(p)
        $$

        其中 V 是那 2f+1 个 VIEW-CHANGE 的集合，O 是刚刚生成的所有 PRE-PREPARE 列表。

4. **其他副本验证 NEW-VIEW 并进入新视图**

   任何副本 j 收到 NEW-VIEW(v+1, V, O) 时：

   - 验证 NEW-VIEW 签名；
   - 验证 V 中每条 VIEW-CHANGE 的签名和结构；
   - 自己根据 V 再算一遍 min-s、max-s 和对应的 PRE-PREPARE 集合 O'；
   - 检查 O' 是否与 NEW-VIEW 中的 O 完全一致（防止 primary 篡改）；
   - 若一切通过：
     - 把 V 和 O 记入日志；
     - 若 min-s 大于自己原来的 stable checkpoint 序号，则更新自己的 stable checkpoint 并丢弃更早的日志；
     - 对 O 中每一条 PRE-PREPARE(v+1, s, d)：
       - 像平时一样进入 prepare 阶段：记录日志并广播 PREPARE(v+1, s, d)。

   从此之后，大家就在新视图 v+1 下继续正常的 pre-prepare / prepare / commit 流程。


# 改进算法
tendermint, Hotstuff