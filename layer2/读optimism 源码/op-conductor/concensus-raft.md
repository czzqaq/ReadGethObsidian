# raft 
## CFT

![[Pasted image 20251002214755.png]]


也是一种共识算法。
我之前接触到的，是 BFT 模型，而现在的Raft 是CFT (crash faulty tolerant)。关于 CFT vs BFT 见：[Consensus Algorithms in Distributed System - GeeksforGeeks](https://www.geeksforgeeks.org/operating-systems/consensus-algorithms-in-distributed-system/)

大多数内容来自：
[Raft 分布式一致性(共识)算法 论文精读与ETCD源码分析_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1CK4y127Lj?spm_id_from=333.788.videopod.episodes&vd_source=6e55e68e1f669c8455e9ef03f6a56005)


## 选举过程
![[Pasted image 20251003181946.png]]

