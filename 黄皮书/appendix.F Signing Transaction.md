## 附录 F. 签名 Transactions

**Transactions** 使用可恢复的 ECDSA 签名进行签名。该方法使用由 Courtois 等人（2014）描述的 **SECP-256k1** 椭圆曲线，并类似于 Gura 等人（2004）在第 9 页第 3 段所描述的方式实现。

我们假设发送者拥有一个有效的私钥 $pr$，它是一个在区间 $[1, \text{secp256k1n} - 1]$ 中随机选择的正整数（以大端格式的 32 字节数组形式表示）。

我们假设存在如下函数：**ECDSAPUBKEY**, **ECDSASIGN**, 和 **ECDSARECOVER**。这些函数在文献中已有形式化定义，例如 Johnson 等人（2001）所述。

定义如下：

$$
\begin{aligned}
\text{ECDSAPUBKEY}(p_r \in \mathbb{B}_{32}) &\equiv p_u \in \mathbb{B}_{64} \\
\text{ECDSASIGN}(e \in \mathbb{B}_{32}, p_r \in \mathbb{B}_{32}) &\equiv (v \in \mathbb{B}_1, r \in \mathbb{B}_{32}, s \in \mathbb{B}_{32}) \\
\text{ECDSARECOVER}(e \in \mathbb{B}_{32}, v \in \mathbb{B}_1, r \in \mathbb{B}_{32}, s \in \mathbb{B}_{32}) &\equiv p_u \in \mathbb{B}_{64}
\end{aligned}
$$

其中：

- $p_u$ 是公钥，为大小为 64 字节的字节数组（由两个小于 $2^{256}$ 的正整数拼接而成）。
- $p_r$ 是私钥，为大小为 32 字节的字节数组（或前述区间内的一个正整数）。
- $e$ 是该 **Transaction** 的哈希，记作 $h(T)$。
- $v$ 是“恢复标识符”，它是一个字节，指定椭圆曲线点中 $r$ 为 x 值所对应点的坐标的奇偶性和有限性。其值在区间 $[0, 3]$，但我们声明值为 2 或 3（表示无限值）为无效。值 0 表示 y 坐标为偶数，1 为奇数。

我们声明一个 ECDSA 签名仅在以下所有条件都为真时才是有效的：

$$
\begin{aligned}
0 &< r < \text{secp256k1n} \\
0 &< s < \frac{\text{secp256k1n}}{2} + 1 \\
v &\in \{0, 1\}
\end{aligned}
$$

其中：

$$
\text{secp256k1n} = 115792089237316195423570985008687907852837564279074904382605163141518161494337
$$

请注意，对 $s$ 的限制比 **ΞECREC** 预编译合约中的限制更严格；详见 Buterin（2015）在 **EIP-2** 中的讨论。

对于给定的私钥 $p_r$，其对应的以太坊地址 $A(p_r)$（160 位）定义为对应 **ECDSA** 公钥的 Keccak-256 哈希的最低 160 位：

$$
A(p_r) = B^{96..255}(\text{KEC}(\text{ECDSAPUBKEY}(p_r)))
$$

消息哈希 $h(T)$ 是该 **Transaction** 的 Keccak-256 哈希。签名方式有四种变体，定义如下：


$$
L_X(T) \equiv
\begin{cases}
(T_n, T_p, T_g, T_t, T_v, \mathbf{p}) & \text{if } T_x = 0 \land T_w \in \{27, 28\} \\
(T_n, T_p, T_g, T_t, T_v, \mathbf{p}, \beta, (), ()) & \text{if } T_x = 0 \land T_w \in \{2^\beta + 35, 2^\beta + 36\} \\
(T_c, T_n, T_p, T_g, T_t, T_v, \mathbf{p}, T_A) & \text{if } T_x = 1 \\
(T_c, T_n, T_f, T_m, T_g, T_t, T_v, \mathbf{p}, T_A) & \text{if } T_x = 2
\end{cases}
$$

其中：

$$
\mathbf{p} \equiv
\begin{cases}
T_i & \text{if } T_t = \emptyset \\
T_d & \text{otherwise}
\end{cases}
$$

消息哈希 $h(T)$ 定义如下：

$$
h(T) \equiv
\begin{cases}
\text{KEC}(\text{RLP}(L_X(T))) & \text{if } T_x = 0 \\
\text{KEC}(T_x \cdot \text{RLP}(L_X(T))) & \text{otherwise}
\end{cases}
$$

签名后的 **Transaction** $G(T, p_r)$ 定义如下：

$$
G(T, p_r) \equiv T \quad \text{except:}
$$

$$
(T_y, T_r, T_s) = \text{ECDSASIGN}(h(T), p_r)
$$

重申：

$$
T_r = r 
$$
$$
T_s = s
$$

而旧格式的 **Transaction** 中的 $T_w$ 为 $27 + T_y$ 或 $2^{\beta} + 35 + T_y$。

我们进一步定义 **Transaction** 的发送者函数 $S(T)$ 为：

$$
S(T) \equiv B^{96..255}(\text{KEC}(\text{ECDSARECOVER}(h(T), v, T_r, T_s)))
$$

其中：

$$
v \equiv
\begin{cases}
T_w - 27 & \text{if } T_x = 0 \text{ and } T_w \in \{27, 28\} \\
(T_w - 35) \bmod 2 & \text{if } T_x = 0 \text{ and } T_w \in \{2^{\beta} + 35, 2^{\beta} + 36\} \\
T_y & \text{if } T_x = 1 \text{ or } T_x = 2
\end{cases}
$$

我们断言：一个签名后的 **Transaction** 的发送者就是签名者的地址，这是显然成立的：

$$
\forall T, \forall p_r: \quad S(G(T, p_r)) \equiv A(p_r)
$$


# SECP-256k1
  
## 一些思考
### RSA
这里先从最简单的非对称加密开始理解

1. 选择两个大素数 \( p, q \)，计算 \( n = pq \)。 
2. 计算欧拉函数 \( \phi(n) = (p-1)(q-1) \)。
3. 选择整数 \( e \)，使得 \( \gcd(e, \phi(n)) = 1 \)。 
4. 计算 \( d \equiv e^{-1} \mod \phi(n) \)。 公钥为 \( (e, n) \)，私钥为 \( d \)。 加密：\( c = m^e \mod n \) 解密：\( m = c^d \mod n \)


### 椭圆曲线

签名要满足：已知无数多次的签名S，也极其难推断出来私钥d。非对称签名用的都是类似于离散模域对数难以求的问题，因为指数要绕好几个圈，所以算指数容易，而对数的单调性已经被破坏了，不能用逼近的方法求根。函数本身是非线性的，所以也不可能相加减找一个 d 的明确的方程出来。

椭圆曲线系的非对称加密，是用椭圆曲线上的加法定义，以及加很多次最终肯定会绕回去，得到了一个指数计算的**同构**。

指数能绕一圈是因为费马小定理。比如最基础的非对称加密 RSA 。

椭圆曲线上比较显然吧：首先域的大小有限，也就是你一个数自己不停的加，总会回到之前的结果。现在的问题是，为什么加法总是会绕一个恰到好处的环，回到O，而不是先走一段距离，变成一个混循环小数的感觉呢？因为加法满足结合律和交换律（这个好证），所以出现在任何地方的加法都是地位等同的。这当然是非常不严谨的数学证明，不过我就这样理解了。于是，我们定义 Q = dG，其中Q是公钥，G是基点，d是私钥。已知私钥求公钥很好求，而已知公钥求私钥非常困难。

### 交互式验证
#### 过程
![[Pasted image 20250413212709.png]]

1. 被验证方生成一对临时的公私钥。把公钥发给认证方，作为 commitment。
2. 认证方随机生成一个随机数 r，称为challenge。
3. 被认证方将自己的私钥、r、防止重入的随机数k，产生一个签名，发给认证方。此外认证方还知道临时的commitment 公钥，以及被认证方分享的公钥Q。

![[Pasted image 20250413213248.png]]

#### 思考
显然，签名的过程需要不可重复，也就是攻击者重放签名的报文没意义，于是，肯定要引入一个随机数 ，作为题目，每次验证都是新的。

可以预见的，签名起码是 r 和 d 的函数。实际的签名总是还有一个未知数k，它是生成的临时私钥。因为r 是公开的，所以如果同样的r 题目，就可以用重放攻击了，所以引入一个k 以后，无论如何攻击者都不知道这个隐藏的随机数是多少，题目都看不全。


### fiat-shamir 变换

类似的，区别在于，现在 r 是被认证方生成的值，它将和s 一同发送给认证方。
![[Pasted image 20250413214411.png]]
![[Pasted image 20250413214420.png]]
#### 思考

对于非交互式验证，区别在于 r 也由被验证者生成。这样，当然没有办法防止重放攻击了。单一的fiet shamir 变换并不能防止重放攻击，往往需要结合时间戳、nonce 等。而web3 签名验证确实有nonce，重放之后用的还是之前的nonce，所以这样的重放是无效的。
那为什么还非得有 r 不可？因为要对d 做包装，而且这个包装一定要是随机的，不然攻击者总能试出来d（就好像是密码要加盐一样）。

签名的过程和它已经很接近了，但是签名是针对消息的，于是还有一个参与计算的参数。

## ECDSA

### ECDSA-SIGN

签名过程如下：

1. 选择一个随机数 $k \in [1, n-1]$
2. 计算椭圆曲线点 $R = kG$，取 $r = R_x \bmod n$
3. 计算签名值：

$$
s = k^{-1}(z + rd) \bmod n
$$

其中：
- $z$ 是消息的哈希
- $d$ 是私钥
- $G$ 是椭圆曲线基点

最终的签名为一对数值 $(r, s)$。


### ECDSA-VERIFY

验证过程如下：

1. 检查 $r, s \in [1, n - 1]$，否则拒绝
2. 计算：

$$
w = s^{-1} \bmod n
$$

$$
u_1 = zw \bmod n,\quad u_2 = rw \bmod n
$$

$$
(x_1, y_1) = u_1G + u_2Q
$$

3. 验证是否成立：

$$
r \equiv x_1 \bmod n
$$

如果相等，则签名有效。


### ECDSA-RECOVER （公钥恢复）

在某些协议中（如以太坊签名格式），我们可以直接从签名 $(r, s)$ 和消息哈希 $z$ 中恢复出公钥 $Q$，前提是多提供一个恢复参数 $v$（表明椭圆曲线点的某个解）。

思路如下：

1. 根据 $r$ 和 $v$ 恢复出椭圆曲线点 $R$（可能有两个候选点）
2. 利用等式：

$$
sR = zG + rQ
$$

等价于：

$$
rQ = sR - zG
$$

从而可得：

$$
Q = r^{-1}(sR - zG)
$$

3. 验证恢复出的 $Q$ 是否与预期匹配。如果匹配，则恢复成功。
