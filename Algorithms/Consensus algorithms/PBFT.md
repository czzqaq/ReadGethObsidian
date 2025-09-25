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

# 具体行为

## 角色
### 客户端
负责决定主节点。负责控制请求间隔。
向主节点发送请求系统应答的 request，并追踪全部节点的应答。最终决定一致的consensus结果。

### 主节点
pre-prepare 阶段负责广播来自 客户端 的request。其他阶段等于一个普通节点。

## 流程
![[Pasted image 20250925152120.png]]
- **request**: client 给主节点发请求。
- **预准备（pre-prepare）**：主节点为请求分配序列号n，多播PRE-PREPARE消息（携带请求摘要）给备份，并将消息记入日志。
- **准备（prepare）**：备份收到并验证PRE-PREPARE消息后，发PREPARE消息给所有副本，并将消息记入日志。
- **提交（commit）**：副本（含主节点）收到足够（2f+1）PREPARE消息后，多播COMMIT消息。
- **reply**: 收到2f+1个匹配COMMIT消息后，副本提交请求。
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

