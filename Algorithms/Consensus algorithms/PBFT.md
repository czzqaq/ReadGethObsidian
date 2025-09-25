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


### 为什么 3f+1
这个算是基础理论了，在很多
可以看论文[Asynchronous Consensus and Broadcast Protocols](https://dl.acm.org/doi/pdf/10.1145/4221.214134)