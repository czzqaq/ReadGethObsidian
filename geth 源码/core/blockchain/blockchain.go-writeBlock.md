
# writeKnownBlock 
## 说明

在[[blockchain.go-insertChain]] 中调用，如果snapshot 中存在某个block，但是blockchain 中没有它，就会进入该逻辑。


## 代码

```go
// writeKnownBlock updates the head block flag with a known block
// and introduces chain reorg if necessary.
func (bc *BlockChain) writeKnownBlock(block *types.Block) error {
	current := bc.CurrentBlock()
	if block.ParentHash() != current.Hash() {
		bc.reorg(current, block.Header())
	}
	bc.writeHeadBlock(block)
	return nil
}

```
### writeHeadBlock
把一个 chain head 添加到现在的chain 中，假定它确实就是head。涉及到了rawdb 的更新，current block 重新设置。

# writeBlockWithState

# 说明

## 内容

更新rawdb 的 block 之外，还更新了 Receipt 和 block chain 上关联的world state trie。


# writeBlockAndSetHead
## 说明

先调用 writeBlockWithState，再调用 reorg, 最后调用writeHeadBlock 修改blockhead。


