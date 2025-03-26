# 说明

是对[[leveldb]] 的应用。

用来保存区块链相关的数据，例如全部历史的区块头等。



## 使用例

```go
  
// Commit writes the block and state of a genesis specification to the database.
// The block is committed as the canonical head block.
func (g *Genesis) Commit(db ethdb.Database, triedb *triedb.Database) (*types.Block, error) {
    // ... other code
	batch := db.NewBatch()
	rawdb.WriteGenesisStateSpec(batch, block.Hash(), blob)
	rawdb.WriteBlock(batch, block)
	rawdb.WriteReceipts(batch, block.Hash(), block.NumberU64(), nil)
	rawdb.WriteCanonicalHash(batch, block.Hash(), block.NumberU64())
	rawdb.WriteHeadBlockHash(batch, block.Hash())
	rawdb.WriteHeadFastBlockHash(batch, block.Hash())
	rawdb.WriteHeadHeaderHash(batch, block.Hash())
	rawdb.WriteChainConfig(batch, block.Hash(), config)
	return block, batch.Write()
}
```


## freeze
`freezer` 用于存储 **不常访问的历史数据**，减少对 LevelDB 的读写压力。
这部分是rawdb 的主要作用。不然直接调用leveldb 就行了对吧。

