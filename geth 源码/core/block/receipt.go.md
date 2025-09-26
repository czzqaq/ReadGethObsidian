# type Receipt struct

```go
// Receipt represents the results of a transaction.
type Receipt struct {
	// Consensus fields: These fields are defined by the Yellow Paper
	Type              uint8  `json:"type,omitempty"`
	PostState         []byte `json:"root"`
	Status            uint64 `json:"status"`
	CumulativeGasUsed uint64 `json:"cumulativeGasUsed" gencodec:"required"`
	Bloom             Bloom  `json:"logsBloom"         gencodec:"required"`
	Logs              []*Log `json:"logs"              gencodec:"required"`

	// Implementation fields: These fields are added by geth when processing a transaction.
	TxHash            common.Hash    `json:"transactionHash" gencodec:"required"`
	ContractAddress   common.Address `json:"contractAddress"`
	GasUsed           uint64         `json:"gasUsed" gencodec:"required"`
	EffectiveGasPrice *big.Int       `json:"effectiveGasPrice"` // required, but tag omitted for backwards compatibility
	BlobGasUsed       uint64         `json:"blobGasUsed,omitempty"`
	BlobGasPrice      *big.Int       `json:"blobGasPrice,omitempty"`

	// Inclusion information: These fields provide information about the inclusion of the
	// transaction corresponding to this receipt.
	BlockHash        common.Hash `json:"blockHash,omitempty"`
	BlockNumber      *big.Int    `json:"blockNumber,omitempty"`
	TransactionIndex uint        `json:"transactionIndex"`
}
```

## 黄皮书相关
### 数据
在 [[4.4.1 Transaction Receipt]] 中，有理论的部分。

#### Type 
交易的type，比如 Dynamic Fee 对应的 `2` 

#### Status
交易状态码。类型确实是uint64，但其实只有
```go
const (
	ReceiptStatusFailed = uint64(0)
	ReceiptStatusSuccessful = uint64(1)
)
```
注：在pre-byzanting 时期，`PostState` 替代了 State 字段，它是交易执行后的 world state 根哈希.现在只有 Status 了。
#### CumulativeGasUsed
该交易执行完成后，所在 Block 的 累计 Gas 消耗。

#### Logs
交易过程中产生的日志集合。

#### Bloom
一个Bloom Filter 的buffer。

### 日志
这部分其实是在 `types/log.go` 中的内容，不过很少，就放在这里了。

#### 日志结构
一个log 集合中的log 由：
一个合约地址+若干event+数据 组成。
```go
type Log struct {
	// Consensus fields:
	// address of the contract that generated the event
	Address common.Address `json:"address" gencodec:"required"`
	// list of topics provided by the contract.
	Topics []common.Hash `json:"topics" gencodec:"required"`
	// supplied by the contract, usually ABI-encoded
	Data []byte `json:"data" gencodec:"required"`

```
注意，Topics 的type 是Hash 的数组。联想到 做 topic 的field 必须是可哈希的。这里保存的就是哈希值。

### Bloom 计算
详见 [[bloom9.go]]


# receipt 来源
这里主要讲把Transaction execution 结果打包为receipt 的过程。
前置知识：[[6. Transaction Execution]]

