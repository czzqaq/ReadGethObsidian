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

### 为什么 2f+1
因为假设收到了 f 个恶意的节点的回应，那么起码多数的 f+1 还是诚实的。

### 为什么 f+1
client  收到 **f+1** 个相同回复，即可确认。
这是因为最多有 f 个节点捣乱，至少有一个诚实的节点在prepare 阶段确定了结果的真实性。




