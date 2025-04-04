
# 说明

这里我们关注的是 BlockChain struct 中的如下几个字段, 特别是 `currentBlock`  的来源和含义。
```go
    currentBlock      atomic.Pointer[types.Header] // Current head of the chain
    currentSnapBlock  atomic.Pointer[types.Header] // Current head of snap-sync
    currentFinalBlock atomic.Pointer[types.Header] // Latest (consensus) finalized block
    currentSafeBlock  atomic.Pointer[types.Header] // Latest (consensus) safe block
```


## 定义
参考 `BlockChainReader.go` 中如下四个函数的使用场景和注释：
```go
// CurrentBlock retrieves the current head block of the canonical chain. The
// block is retrieved from the blockchain's internal cache.
func (bc *BlockChain) CurrentBlock() *types.Header {
	return bc.currentBlock.Load()
}
// CurrentSnapBlock retrieves the current snap-sync head block of the canonical
// chain. The block is retrieved from the blockchain's internal cache.
func (bc *BlockChain) CurrentSnapBlock() *types.Header {
	return bc.currentSnapBlock.Load()
}
// CurrentFinalBlock retrieves the current finalized block of the canonical
// chain. The block is retrieved from the blockchain's internal cache.
func (bc *BlockChain) CurrentFinalBlock() *types.Header {
	return bc.currentFinalBlock.Load()
}
// CurrentSafeBlock retrieves the current safe block of the canonical
// chain. The block is retrieved from the blockchain's internal cache.
func (bc *BlockChain) CurrentSafeBlock() *types.Header {
	return bc.currentSafeBlock.Load()
}
```

从注释中就能明白：
- `CurrentBlock()` 指的是 Canonical BlockChain 的顶端。这个Canonical 是根据本地储存定的，和 concensus 无关。比如在成功新插入（[[blockchain.go-insertChain]]）一个 block 后，`CurrentBlock()` 就**一定**会改变，因为本地肯定会先reorg，然后把新block 插入到Canonical chain 。
- `CurrentFinalBlock()` ，FinalBlock 比较好理解，就是在共识中被Casper FFG finalize 的那个。
- `CurrentSafeBlock()` SafeBlock 是一个geth 的内部概念，用来标识**运行时**“在某个高度上我们认为足够安全的区块”。在回滚的时候被使用。为了更好的理解 safeBlock 这个概念，可以参考 [[#loadLastState()]] 
- `CurrentSnapBlock()` 很多时候都跟 `CurrentBlock()` 在一起被赋值，不过它的使用场景是在 [[eth]] / downloader 中。snap sync 是一种新的同步区块机制，传统的full sync 中，是每个block 都重放 Transaction Execution，而 snapsync，则是先从某个节点下载状态快照，向后验证，向前同步。所谓的snapBlock，是已经下载了的block 快照，它的安全性无法得到充分保障。

# CurrentBlock()

## 使用
### reorg
[[blockchain.go-reorg]] 传入的参数是：
```go
func (bc *BlockChain) reorg(oldHead *types.Header, newHead *types.Header) error
```



# loadLastState()

# setHeadBeyondRoot