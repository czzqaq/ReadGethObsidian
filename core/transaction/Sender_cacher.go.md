
在[[blockchain.go-insertChain]] 中第一次提及：
```go
// in blockchain.go
// Start a parallel signature recovery (signer will fluke on fork transition, minimal perf loss)
SenderCacher().RecoverFromBlocks(
    types.MakeSigner(bc.chainConfig, chain[0].Number(), chain[0].Time()),
    chain
)
```
