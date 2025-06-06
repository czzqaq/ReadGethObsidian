
**Hex-prefix encoding** 是一种高效的将任意数量的半字节（nibble）编码为字节数组的方法。在 **trie**（前缀树）这一唯一使用场景中，它还能够存储一个额外的标志位，用于区分不同的节点类型。

它被定义为函数 $\text{HP}$，该函数将一串半字节 x（属于集合 $\mathbb{Y}$）和一个布尔值映射为一串字节（属于集合 $\mathbb{B}$）：
还有输入t，它只有0 和 1两种取值，当是leaf 时，t=1，当时extension 时，t=0
x 是nibbles，x 的取值范围只有 0-15
f(t) 表示计算prefix。


即：

- 若 $|x|$ 为偶数：

  $$
  \text{HP}(x, t) = (16f(t), 16x[0] + x[1], 16x[2] + x[3], \ldots)
  $$

- 若 $|x|$ 为奇数：

  $$
  \text{HP}(x, t) = (16(f(t) + 1) + x[0], 16x[1] + x[2], 16x[3] + x[4], \ldots)
  $$

其中函数 $f(t)$ 定义如下：

$$
f(t) = 
\begin{cases}
2 & \text{if } t \ne 0 \\
0 & \text{otherwise}
\end{cases}
$$





因此，第一个字节的高半字节包含两个标志位：

- 最低位表示长度的奇偶性；
- 次低位表示标志 **t**。

若半字节数量为偶数，则第一个字节的低半字节为 0；若为奇数，则为第一个半字节的值。其余所有半字节（此时为偶数个）将被正确地编码进后续的字节中。