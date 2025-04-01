
以太坊交易中，发送者地址是从数字签名中恢复得出的。这是一个涉及 ECDSA 的操作，计算较重。因此，如果你有很多交易（比如从区块中加载的），你可能希望先将它们的发送者信息预先计算好并缓存，以避免后续重复执行 `ecrecover` 操作。

> **What is ecrecover?**
  ecrecover is a crucial function in Ethereum that recovers the address that signed a message.


`Sender_cacher.go` 为一批交易（`*types.Transaction`）或区块（`*types.Block`）中的交易 **异步并发执行 `types.Sender(signer, tx)`**，从交易签名中恢复（[[transaction_signing.go#signer]]) 发送者地址，并将其缓存，供后续使用。

# 相关
在[[blockchain.go-insertChain]] 中第一次提及：
```go
// in blockchain.go
// Start a parallel signature recovery (signer will fluke on fork transition, minimal perf loss)
SenderCacher().RecoverFromBlocks(
    types.MakeSigner(bc.chainConfig, chain[0].Number(), chain[0].Time()),
    chain
)
```
