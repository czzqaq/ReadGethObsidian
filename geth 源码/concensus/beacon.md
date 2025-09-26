# 小代码
## isPostMerge
```go
func isPostMerge(config *params.ChainConfig, blockNum uint64, timestamp uint64) bool {
    mergedAtGenesis := config.TerminalTotalDifficulty != nil && config.TerminalTotalDifficulty.Sign() == 0
    return mergedAtGenesis ||
        config.MergeNetsplitBlock != nil && blockNum >= config.MergeNetsplitBlock.Uint64() ||
        config.ShanghaiTime != nil && timestamp >= *config.ShanghaiTime
}
```
通过这两个条件认为进入了ETH2，**或**的关系
1. difficulty 为 0，说明不是POW
2. block number 大于分叉的高度，且block 的产生时间晚于 `ShanghaiTime`

## Author
```go
// Author implements consensus.Engine, returning the verified author of the block.
func (beacon *Beacon) Author(header *types.Header) (common.Address, error) {
    ... 
    return header.Coinbase, nil
}
```
block 的author 是其beneficiary。


## fianalize
处理withdraw 部分，按照withdraw 中的amount ，给address 的statedb 中增加 balance 的值。
对应 [[12 Block Finalisation#12.1 执行 **Withdrawals**]]

# VerifyHeader
这部分内容对应了 [[4.4.4 Block header validility]] 
将《黄皮书》中对区块头有效性函数 $V(H)$ 的规范，与 Go 代码中 `VerifyHeader()` 和 `verifyHeader()` 的实现相对照：

##   黄皮书

黄皮书中定义了区块头有效性验证函数 $V(H)$：

$$
\begin{aligned}
V(H) \equiv \ & H_g \leq H_l \ \land \\
& H_l < P(H)_{H_l'} + \left\lfloor \frac{P(H)_{H_l'}}{1024} \right\rfloor \ \land \\
& H_l > P(H)_{H_l'} - \left\lfloor \frac{P(H)_{H_l'}}{1024} \right\rfloor \ \land \\
& H_l \geq 5000 \ \land \\
& H_s > P(H)_{H_s} \ \land \\
& H_i = P(H)_{H_i} + 1 \ \land \\
& \Vert H_x \Vert \leq 32 \ \land \\
& H_f = F(H) \ \land \\
& H_o = \text{KEC}(\text{RLP}(())) \ \land \\
& H_d = 0 \ \land \\
& H_n = \text{0x0000000000000000} \ \land \\
& H_a = \text{PREVRANDAO()}
\end{aligned}
$$

##   与代码实现逐项对照分析

| 黄皮书字段                  | 规则说明                           | `verifyHeader` 中实现位置                                 |
| ---------------------- | ------------------------------ | ---------------------------------------------------- |
| $H_x$                  | `extraData` 最长 32 字节           | `len(header.Extra) > MaximumExtraDataSize`           |
| $H_n$                  | `nonce = 0x0000000000000000`   | `header.Nonce != beaconNonce`                        |
| $H_o$                  | `ommerHash = KEC(RLP(()))`     | `header.UncleHash != types.EmptyUncleHash`           |
| $H_s$                  | `timestamp > parent.timestamp` | `header.Time <= parent.Time`                         |
| $H_d$                  | `difficulty = 0`               | `header.Difficulty != beaconDifficulty`              |
| $H_g \leq H_l$         | `gasUsed <= gasLimit`          | `header.GasUsed > header.GasLimit`                   |
| $H_l \leq MaxGasLimit$ | `gasLimit ≤ 2^63-1`            | `header.GasLimit > MaxGasLimit`                      |
| $H_i = P(H)_i + 1$     | `block number = parent + 1`    | `Sub(header.Number, parent.Number) != 1`             |
| $H_f = F(H)$           | base fee 验证（EIP-1559）          | `eip1559.VerifyEIP1559Header(...)`                   |
| $H_a = PREVRANDAO()`   | `prevRandao` 验证                | ❌ **未显式验证**，但可能在其他地方处理                               |
| $H_l$ 合法性              | gas limit 波动范围限制               | `eip1559.VerifyEIP1559Header(...)`->`VerifyGaslimit` |
| withdrawalsHash        | 上海升级后必须存在                      | ✅ `header.WithdrawalsHash == nil`                    |
| EIP-4844 字段（Cancun）    | 合规性检查                          | ✅ `header.ParentBeaconRoot` 等                        |

### 未提及的 prevRandao 合法性
prevRandao 字段是上一个slot 中，beacon 链上打包的一个随机数。
见[[beacon]] ，这个目录下实现了和外界的beacon 交互的逻辑。beacon 中的 `package engine` 也被[[miner]] 使用，所以我大胆推测，这个 block 字段是在打包时才最终确定的。之所以写成这样，恐怕也是beacon 链上的内容在这个地方太难获得了。

## 额外逻辑：PoS 与 PoW 的区分

在 `VerifyHeader` 中：

```go
if header.Difficulty.Sign() > 0 {
    return beacon.ethone.VerifyHeader(...)
}
```

说明：
- 如果 `difficulty > 0`，视为 PoW 时代，交由原始共识引擎处理。
- 如果 `difficulty == 0`，视为 PoS 区块，使用 `verifyHeader()` 进行 PoS 规则校验。

此外还有一个防止“倒退”的判断：

```go
if parent.Difficulty.Sign() == 0 && header.Difficulty.Sign() > 0 {
    return ErrInvalidTerminalBlock
}
```

防止从 PoS 拿着一个 PoW 区块“篡改链”。

