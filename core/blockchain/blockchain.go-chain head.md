#archived 
# CurrentBlock()

## 说明

这里我们关注的是 BlockChain struct 中的如下几个字段, 特别是 `currentBlock`  的来源和含义。
```go
    currentBlock      atomic.Pointer[types.Header] // Current head of the chain
    currentSnapBlock  atomic.Pointer[types.Header] // Current head of snap-sync
    currentFinalBlock atomic.Pointer[types.Header] // Latest (consensus) finalized block
    currentSafeBlock  atomic.Pointer[types.Header] // Latest (consensus) safe block
```
### 定义
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


## 来源

### loadLastState 

详见用rawdb 中写入的current block 赋值。

rawdb 中，HeadBlockHash，也就是rawdb 中保存的Current block 是来源是：
- setHeadBeyondRoot
- writeHeadBlock
- genesis.Commit()

没什么可关注的地方。 

### setHeadBeyondRoot
见[[#setHeadBeyondRoot()]]

### writeHeadBlock
详见 [[blockchain.go-writeBlock#writeKnownBlock]] ，这个函数是主要的CurrentBlock 更新者。

### ResetWithGenesisBlock
把CurrentBlock 设置为 genesis block

# setHeadBeyondRoot()

## 作用
在创建新的block 时（`blockchain.go: newBlockChain`)调用，注意到newBlockChain 的效果就是来从硬盘 中取出一条BlockChain。

为了更好的说明这个函数的作用，必须先说明以下的概念。
### frozen

在以太坊节点中，为了提高性能、减少内存和数据库负担，**Geth 引入了“冷/热数据分离存储”机制**。其中：

- **Hot（热）数据**：最近的区块、状态等，频繁读写，存储在 **LevelDB** 中（活跃数据库）。
- **Frozen（冷）数据**：老旧区块、收据、头部、体等，写入后不再修改，存储在 **Freezer** 中。
除非发生链回滚，否则frozen 区块不会被修改。
### rewind

流程见 `BlockChain.rewindHashHead()` ，以及 `HeaderChain.setHead()` 可以概括为从 Current head 开始，一直寻找其 parent，直到找到拥有正确 state 的Block才停止，rewind 可以一直持续到 genesis。这会使得BlockChain 的height降低，这个找到的正确block 会成为新的current block，而此后的block 会被删除。

### repair flag

```go
func (bc *BlockChain) setHeadBeyondRoot(head uint64, time uint64, root common.Hash, repair bool) (uint64, error) {}
```

其中的repair 变量指的是对rewind 的激进程度的描述。rewind可能是：

- 【一定发生】 设置更低高度的为current head。响应的修改raw db 中的current hash等。
- 【当可置信的block 的高度小于 frozen block 的高度时】 删除高度高于置信的block 的部分。涉及到非常剧烈的rawdb 的修改。

而repair 模式下，`setHeadBeyondRoot` 函数会尽量避免第二个删除情况的发生。

### 调用的时机


它调用的时机有：
1. NewBlockChain 调用时，当 [[#loadLastState()]] 没有正确恢复 Current head 的state。此处调用的repair flag是true，倾向于仅仅是把没有加载到的state 重新读出来。
2. NewBlockChain 调用时，当 frozen database 中的block height 高于Current block 的number。类似的，也会做rewind 操作，但区别在于，此时链本身的组织就不对，因此做正常的回滚操作。

## 部分函数
### loadLastState() 
作用是从rawdb 中恢复数据。恢复的内容包括：CurrentBlock, currentSnapBlock, currentFinalBlock。

### db.ancients
也就是返回frozen 的高度
```go
// Ancients returns the length of the frozen items.
func (f *Freezer) Ancients() (uint64, error) {
    return f.frozen.Load(), nil
}
```

# reorg

## 说明

### 调用时机

reorg 在 [[#SetCanonical]]，[[blockchain.go-writeBlock#writeKnownBlock]]， [[blockchain.go-writeBlock#writeBlockAndSetHead]] 这三个函数中被调用，它们的共性是都可能发生Chain head 的变化.

### 作用
当链发生重组时：
- 找到旧链（Old Chain）和新链（New Chain）的**共同祖先（common ancestor）**
- **撤销**旧链上的区块（undo）
- **应用**新链上的区块（apply）
- **更新交易索引与日志**
- **保持链状态一致性**

其中，旧 chain head 是指当前链的head，一般通过 `bc.CurrentHead()` 得到，而 new head 则是要替换的chain head，它位于另一个分叉。


## 过程
### 对齐链高度
```go
    newChain    []*types.Header
    oldChain    []*types.Header
    
	// Reduce the longer chain to the same number as the shorter one
	if oldHead.Number.Uint64() > newHead.Number.Uint64() {
		// Old chain is longer, gather all transactions and logs as deleted ones
		for ; oldHead != nil && oldHead.Number.Uint64() != newHead.Number.Uint64(); oldHead = bc.GetHeader(oldHead.ParentHash, oldHead.Number.Uint64()-1) {
			oldChain = append(oldChain, oldHead)
		}
	} else {
		// 类似的，把new head 赋值为和old head 高度一致的那个，同时把退下来的记录到newChain 中
	}
```

### 找到共同祖先
```go
for {
	if oldHead.Hash() == newHead.Hash() {
		commonBlock = oldHead
		break
	}
	// 继续回退两条链，直到 hash 相等
}
```

### 撤销旧链
```go
	// Undo old blocks in reverse order
	for i := 0; i < len(oldChain); i++ {
		// Collect all the deleted transactions
		block := bc.GetBlock(oldChain[i].Hash(), oldChain[i].Number.Uint64())

		}
		for _, tx := range block.Transactions() {
			deletedTxs = append(deletedTxs, tx.Hash())
		}
		
		// Collect deleted logs and emit them for new integrations
		if logs := bc.collectLogs(block, true); len(logs) > 0 {
			// Hook into the reverse emission part
		}
	}
```

### 应用新链
```
for block in newChain (from oldest to newest):
	- 收集新增交易
	- 收集并发送新增日志
	- 更新当前 head block
```
和上面的类似，产生 rebirthTxs 和 rebirthLogs。实际上要被删除的 tx 是deletedTxs - rebirthTxs

### raw db 更新
包括删除 `deletedTxs - rebirthTxs` 的交易记录。
删除旧链上的 block hash marker，这部分也是无用索引。

