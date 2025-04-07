## 12. 区块最终确定（**Block Finalisation**）

区块的最终确定过程包括三个阶段：

1. 执行 **withdrawals**；
2. 验证 **transactions**；
3. 验证状态（**verify state**）。

---

### 12.1 执行 **Withdrawals**

在处理完区块中的所有 **transactions** 之后，执行对应的 **withdrawals** 操作。

**Withdrawal** 是指将指定的 **Gwei** 金额增加到账户余额中。这不是转账，因为没有其他账户余额被减少，它本质上是资金的“创造”。**Withdrawal** 操作无法失败，也不消耗 **gas**。

我们定义函数 $E$ 作为 **withdrawal** 的状态转换函数：

$$
E(\sigma_w, W) \equiv \sigma_{w+1} \tag{177}
$$

$$
\sigma_{w+1} \equiv \sigma_w \text{ except:} \tag{178}
$$

$$
\sigma_{w+1}[W_r]_b \equiv \sigma_w[W_r]_b + (W_a \times 10^9) \tag{179}
$$

其中：

- $W_r$ 是接收账户地址；
- $W_a$ 是以 **Gwei** 表示的金额。

我们进一步定义 $K$，即区块级别的 **withdrawal** 状态转换函数：

$$
K(\sigma, B) \equiv E(E(\sigma, W_0), W_1), \dots \tag{180}
$$

---

### 12.2 交易验证（**Transaction Validation**）

给定的 **gasUsed** 必须准确反映区块内列出的所有 **transactions**：

$$
B_H^g = \ell(R)_u \tag{181}
$$

其中：

- $B_H^g$ 是区块头中记录的总 **gasUsed**；
- $\ell(R)_u$ 是所有 **transaction** 执行累计 **gas** 的最终值。

---

### 12.3 状态验证（**State Validation**）

现在我们定义函数 $\Gamma$，该函数将区块 $B$ 映射到其初始状态：

$$
\Gamma(B) \equiv
\begin{cases}
\sigma_0 & \text{if } P(B_H) = \emptyset \\
\sigma_i & \text{such that } \text{TRIE}(LS(\sigma_i)) = P(B_H)^r \text{ otherwise}
\end{cases} \tag{182}
$$

其中：

- $\text{TRIE}(LS(\sigma_i))$ 表示状态 $\sigma_i$ 的状态树根节点的哈希；
- $P(B_H)^r$ 是区块头中父区块的状态根；
- 状态树（**trie**）是不可变的数据结构，因此其存储和匹配是高效且简单的。


---

最终我们定义完整的区块转换函数 $\Phi$，它将一个不完整的区块 $B$ 映射为一个完整的区块 $B'$：

$$
\Phi(B) \equiv B' \text{ such that:} \tag{183}
$$

$$
B' = B \text{ except:} \tag{184}
$$

$$
B'^r = \text{TRIE}(LS(K(\Pi(\Gamma(B), B), B)))
$$

---

如前所述，$\Pi$ 是状态转换函数，定义为最终交易执行后得到的状态（基于函数 $\Upsilon$，即 **transaction evaluation** 函数）。

对于每笔 **transaction**，我们定义其状态 $\sigma[n]$ 如下：

$$
\sigma[n] =
\begin{cases}
\Gamma(B) & \text{if } n < 0 \\
\Upsilon(\sigma[n - 1], B_T[n]) & \text{otherwise}
\end{cases} \tag{185}
$$

---

对于累计 **gasUsed** 的表示 $R[n]_u$，我们定义为当前交易的 **gas** 加上前一交易的累计值（或 0）：

$$
R[n]_u =
\begin{cases}
0 & \text{if } n < 0 \\
\Upsilon^g(\sigma[n - 1], B_T[n]) + R[n - 1]_u & \text{otherwise}
\end{cases} \tag{186}
$$

---

对于日志 $R[n]_l$ 和状态码 $R[n]_z$，我们使用函数 $\Upsilon^l$ 和 $\Upsilon^z$ 分别定义为：

$$
R[n]_l = \Upsilon^l(\sigma[n - 1], B_T[n]) \tag{187}
$$

$$
R[n]_z = \Upsilon^z(\sigma[n - 1], B_T[n]) \tag{188}
$$

---

最后，$\Pi$ 作为交易执行后的最终状态定义为：

$$
\Pi(\sigma, B) \equiv \ell(\sigma) \tag{189}
$$

---

至此，我们就完整定义了区块转换机制（在共识之前的部分）。


# 12.3 的解释
## 在代码中
客户端会维护一个本地的 **状态数据库（state database）**，其中：

1. 每个区块执行完后，会把新状态更新到状态数据库中；
2. 并将新的状态树的根哈希（`stateRoot`）写入区块头；
3. 下次验证新区块，只需：
    - 加载上一个区块的状态作为起点；
    - 执行新区块的交易；
    - 计算出新状态树的根哈希；
    - 验证这个哈希是否与新区块头中提供的 `stateRoot` 一致。
详见 [[block_validator.go#ValidateState]] 

## 在黄皮书中
以太坊的区块验证机制中，状态验证不是要求客户端**重新从创世区块推演所有状态**，而是通过两个函数建立了一个**推演-验证模型**，用于定义状态的一致性。

###  向前的函数：`Γ(B)`

- `Γ(B)` 是一个 **向前的函数**，给定区块 `B`，返回其“初始状态”。
    
- 如果是创世区块，则返回初始状态 `σ₀`；否则返回其父区块的最终状态 `σᵢ`（通过状态树的根哈希匹配）。

### 向后的函数：`Φ(B)`

- `Φ(B)` 是一个 **向后的函数**，给定一个区块 `B`，通过执行其交易与 withdrawal 操作，得出其最终状态。
    
- 简化解释：
    - 从 `Γ(B)` 得到初始状态；
    - 调用 $\Pi$ 执行交易（使用交易执行函数 `Υ`）得到交易后的状态；
    - 调用 `K` 执行 withdrawal；
    - 最终构建出新的状态树，并提取其根哈希，得到 `B'^r`。

这个 `Φ(B)` 就是从区块 `B` **推导出它应有的状态根哈希**。

### 状态验证的核心逻辑：

- `Γ(B)` 告诉我们：这个区块是基于哪个状态执行的。
- `Φ(B)` 告诉我们：这个区块执行完后应该得出什么状态。
- 两者结合，就可以从任意可信状态出发，对任意区块进行验证。