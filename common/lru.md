见：
https://zhuanlan.zhihu.com/p/34989978

LRU（Least Recently Used）Cache

**数据结构**：

- **哈希表（HashMap）**：用于快速查找缓存中的数据（O(1) 时间复杂度）。
- **双向链表（Doubly Linked List）**：用于维护访问顺序，最近使用的放在表头，最少使用的放在表尾。表现为一个FIFO 的队列。


查找时，一方面通过哈希定位到要找的值，另一方面把对应的链表中的节点移到最开头。
存放时，如果满了，就把最后一个弹出。
