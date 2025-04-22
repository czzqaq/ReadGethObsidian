
# 黄皮书

**修改版 Merkle Patricia Trie（又称 trie）** 提供了一种持久化数据结构，用于在任意长度的二进制数据（字节数组）之间建立映射关系。它以一种可变的数据结构形式定义，用于在 256 位二进制片段与任意长度二进制数据之间建立映射，通常由数据库实现。

该 trie 的核心目标，也是协议规范中唯一的要求，是能够生成一个表示某一组键值对的单一值(a single value)，该值可以是一个 32 字节的序列，也可以是空字节序列。至于如何存储和维护该 trie 的结构，以实现协议的高效执行，则留给具体实现去决定。
> [!note] 也就是定义如何对node encode
> 

形式上，我们假设输入值 $I$ 是一个包含唯一键的字节序列对的集合：

$$
I = \{(k_0 \in \mathbf{B}, v_0 \in \mathbf{B}), (k_1 \in \mathbf{B}, v_1 \in \mathbf{B}), \dots\}
$$

在处理这样的序列时，我们使用通用的下标记法来表示 tuple 的 key 或 value，因此：

$$
\forall I \in \mathcal{I} : I \equiv (I_0, I_1)
$$
### nibble
任意的字节序列也可以被视为一系列 nibbles（半字节），具体取决于编码顺序；这里我们假设使用大端（big-endian）表示法。因此：

$$
\gamma(I) = \{(k_0' \in \mathbf{Y}, v_0 \in \mathbf{B}), (k_1' \in \mathbf{Y}, v_1 \in \mathbf{B}), \dots\}
$$

其中：

$$
\forall n : \forall i < 2 \cdot \lVert k_n \rVert : k_n'[i] \equiv 
\begin{cases}
\lfloor k_n[i \div 2] \div 16 \rfloor & \text{if } i \text{ is even} \\
k_n[\lfloor i \div 2 \rfloor] \bmod 16 & \text{otherwise}
\end{cases}
$$
### TRIE
我们定义函数 **TRIE**，用于对该结构中的集合进行编码并返回其根节点：

$$
\text{TRIE}(I) \equiv \text{KEC}\left(\text{RLP}(c(I, 0))\right)
$$
其中c 的定义见下方。

类似前缀树（radix tree），从根节点到叶子节点的遍历过程中可以构造出一个完整的键值对。键在遍历过程中逐步累积，每个分支节点贡献一个 nibble。

然而与前缀树不同，当多个键共享前缀或某个键具有唯一后缀时，trie 提供了两种优化节点。因此，在遍历过程中，除了分支节点，每个扩展节点和叶子节点可能会提供多个 nibbles。

trie 中的三种节点如下：

- **Leaf**：由两个元素组成，第一个元素为尚未被前序路径覆盖的 key 的 nibble 序列，使用 hex-prefix 编码，第二个参数设为 1。
- **Extension**：也是两个元素，第一个元素为多个 key 共享的 nibble 子序列，长度大于 1，使用 hex-prefix 编码，第二个参数设为 0。
- **Branch**：由 17 个元素组成，前 16 个是每个可能的 nibble 值对应的位置，第 17 个用于标识这是终止节点，即此处结束了某个完整 key。

只有在必要时才使用 **branch** 节点；不允许存在仅包含单个非零项的 branch 节点。

### 序列化
在构造节点时，我们使用 **RLP** 对结构进行编码。为了减少存储复杂度。

我们使用 **the node composition function** **c** 对该结构进行形式定义：
如下分别定义了 Leaf、extension、Branch 的序列方法

$$
\begin{aligned}
c(\mathcal{I}, i) \equiv 
\begin{cases}
(\text{HP}(I_0[i .. (\lVert I_0 \rVert - 1)], 1), I_1) & \text{if } \lVert \mathcal{I} \rVert = 1 \text{ 且 } \exists I \in \mathcal{I} \\
(\text{HP}(I_0[i .. (j - 1)], 0), n(\mathcal{I}, j)) & \text{if } i \neq j, \text{ 其中 } j = \max\{x : \exists I \in \mathcal{I}, \lVert I \rVert = x \land I_0[0..(x-1)] = I_0[0..(x-1)]\} \\
(u(0), u(1), \dots, u(15), v) & \text{否则， 其中}
\end{cases}
\end{aligned}
$$

$$
\begin{aligned}
u(j) &\equiv n(\{I : I \in \mathcal{I} \land I_0[i] = j\}, i + 1) \\
v &\equiv 
\begin{cases}
I_1 & \text{if } \exists I \in \mathcal{I} \land \lVert I_0 \rVert = i \\
() \in \mathbf{B} & \text{otherwise}
\end{cases}
\end{aligned}
$$
上面：
I_0 是 key(nibbles) ， I_1 是value。
HP  是Hex-Prefix-Encoding
Leaf 是直接给的，其他两种节点均通过递归函数 n 定义。
HP 的定义详见：[[appendix.C Hex-Prefix Encoding]]

关于函数n，具体的，表示 trie 的节点封装函数当节点的 RLP 编码少于 32 字节时直接存储；否则存储其 **Keccak-256** 的哈希，在子节点为空（即叶子节点）时停止。

$$
n(I, i) \equiv 
\begin{cases}
() \in \mathbf{B} & \text{if } I = \emptyset \\
c(I, i) & \text{if } \lVert \text{RLP}(c(I, i)) \rVert < 32 \\
\text{KEC}(\text{RLP}(c(I, i))) & \text{otherwise}
\end{cases}
$$




### D.1. **Trie Database**

对于 trie 存储结构，我们不对哪些数据被存储、哪些不被存储作出强制规定，因为这是具体实现的问题；我们仅定义一个标识函数，用于将键值集合 $I$ 映射为一个 32 字节的哈希值，并断言对于任意集合 $I$，该哈希值是唯一的。尽管这在严格意义上并不绝对正确，但由于 **Keccak** 哈希函数的抗碰撞性，这种近似是可以接受的。

实际上，一个合理的实现不会为每个键值集合都完全重新计算 trie 根哈希。更高效的方式是维护一个由不同 trie 计算出的节点组成的数据库，或者更形式化地说，对函数 $c$ 进行 **memoization**。该方法利用 trie 的结构特点，可以非常高效地存储并检索多个键值集合。

由于节点间存在依赖关系，可以构造出空间复杂度为 $O(\log N)$ 的 **Merkle-proof**，用以证明某个特定叶节点确实存在于具有指定根哈希的 trie 中。