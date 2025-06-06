
## 为什么需要预编译合约（Precompiled Contracts）？

在以太坊中，智能合约通常是用 Solidity 编写并编译成字节码运行在 EVM（以太坊虚拟机）中。但一些操作（如加密算法或大数运算）计算量大、执行复杂，如果用字节码实现效率低、耗费大量 gas。

因此，为了提高性能和节省 gas，以太坊在一些固定地址（如 `0x01` 到 `0x09` 等）**内置了一些高效的原生合约**，这些称为**预编译合约**。它们是用高效的底层语言实现的，可以像普通合约一样调用。

---

##  预编译合约列表及功能概述

以下是 Appendix E 中定义的所有预编译合约及其功能简述：

|编号|合约名|功能|Gas 费用说明|
|---|---|---|---|
|0x01|`ΞECREC`|恢复签名的公钥（`ecrecover`），用于从交易签名中恢复发送者地址|固定 3000 gas|
|0x02|`ΞSHA256`|计算 SHA-256 哈希值|与输入长度有关：`60 + 12 * ⌈len/32⌉`|
|0x03|`ΞRIP160`|计算 RIPEMD-160 哈希值（并补足为 32 字节）|`600 + 120 * ⌈len/32⌉`|
|0x04|`ΞID`|身份函数，输入即输出|`15 + 3 * ⌈len/32⌉`|
|0x05|`ΞEXPMOD`|计算模幂运算（`B^E mod M`），支持任意精度|gas 计算复杂，取决于输入长度和指数|
|0x06|`ΞSNARKV`|zk-SNARK 证明验证函数，用于零知识证明验证|`34000*k + 45000`，其中 k 是输入对数|
|0x07|`ΞBN ADD`|G1 群上的椭圆曲线点加法（用于 zkSNARK）|固定 150 gas|
|0x08|`ΞBN MUL`|G1 群上的标量乘法（点乘）|固定 6000 gas|
|0x09|`ΞBLAKE2 F`|BLAKE2 哈希函数的压缩函数（EIP-152）|输入长度必须是 213 字节，gas 用户提供|

---

##  小结

- 预编译合约是以太坊内置的高性能函数，用于处理诸如加密、哈希、大数运算等任务。
- 这些合约可以像普通合约一样调用，只是它们在底层由客户端直接实现。
- 使用预编译合约可显著降低 gas 成本，提高执行效率。
- 它们在某些应用（如 zkSNARK、身份验证、哈希计算）中是关键组件。