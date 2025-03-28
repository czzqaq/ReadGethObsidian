
## 黄皮书中的数据

### 直接储存
```go
func (tx *DynamicFeeTx) txType() byte           { return DynamicFeeTxType }

func (tx *DynamicFeeTx) chainID() *big.Int      { return tx.ChainID }

func (tx *DynamicFeeTx) accessList() AccessList { return tx.AccessList }

func (tx *DynamicFeeTx) data() []byte           { return tx.Data }

func (tx *DynamicFeeTx) gas() uint64            { return tx.Gas }

func (tx *DynamicFeeTx) gasFeeCap() *big.Int    { return tx.GasFeeCap }

func (tx *DynamicFeeTx) gasTipCap() *big.Int    { return tx.GasTipCap }

func (tx *DynamicFeeTx) gasPrice() *big.Int     { return tx.GasFeeCap }

func (tx *DynamicFeeTx) value() *big.Int        { return tx.Value }

func (tx *DynamicFeeTx) nonce() uint64          { return tx.Nonce }

func (tx *DynamicFeeTx) to() *common.Address    { return tx.To }

```
### 签名密码学
```go
func (tx *DynamicFeeTx) rawSignatureValues() (v, r, s *big.Int) {
    return tx.V, tx.R, tx.S
}

func (tx *DynamicFeeTx) setSignatureValues(chainID, v, r, s *big.Int) {
    tx.ChainID, tx.V, tx.R, tx.S = chainID, v, r, s
}
```

其中，在 graphql/graphql.go 中，定义了

```go
func (t *Transaction) YParity(ctx context.Context) (*hexutil.Big, error) {
    tx, _ := t.resolve(ctx)
    if tx == nil || tx.Type() == types.LegacyTxType {
        return nil, nil
    }
    v, _, _ := tx.RawSignatureValues()
    ret := hexutil.Big(*v)
    return &ret, nil
}
```

说明这里的V 字段就不是传统的V（parity+27)，而是单纯的 yParity，仅仅有01 两种取值。

### encoding
$$
  L_e^T(T) =
  \begin{cases}
  L_T(T) & \text{if } T_x = 0 \\
  (T_x) \cdot \text{RLP}(L_T(T)) & \text{otherwise}
  \end{cases}
  $$
```go
  func (tx *DynamicFeeTx) sigHash(chainID *big.Int) common.Hash {
	return prefixedRlpHash(
		DynamicFeeTxType,
		[]any{
			chainID,
			tx.Nonce,
			tx.GasTipCap,
			tx.GasFeeCap,
			tx.Gas,
			tx.To,
			tx.Value,
			tx.Data,
			tx.AccessList,
		})
}

```

详见：[[4.4.3 Serialization]]
