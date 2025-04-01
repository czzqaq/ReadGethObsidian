#todo 
# 代码逐行
## sanity check

```go
	// Do a sanity check that the provided chain is actually ordered and linked.
	for i := 1; i < len(chain); i++ {
		block, prev := chain[i], chain[i-1]
		if block.NumberU64() != prev.NumberU64()+1 || block.ParentHash() != prev.Hash() {
			// error log and return.
		}
	}
```

直接看代码，需要插入的number 按序排列，且储存的父哈希正确。

## 事件处理
```go
    defer func() {
        if lastCanon != nil && bc.CurrentBlock().Hash() == lastCanon.Hash() {
            bc.chainHeadFeed.Send(ChainHeadEvent{Header: lastCanon.Header()})
        }
    }()
```

原理见：[[feed.go]]
blockProcFeed 和 chainHeadFeed 的事件被类似于 eth/api_backend 的对象订阅，用来监听变化。
其中，blockProcFeed 的作用是提醒有区块在处理。`chainHeadFeed` 的作用比较重要（如上面的代码），表示的事:

`bc.CurrentBlock()` 是当前区块链的叶子节点，比如，正在打包的那个。lastCanon 是被确确实实插入的block 中，height 最大的那个。如果刚刚被插入的那个block 成为了主链的链头，就产生信号`chainHeadFeed`。
## sign transaction
```go
    SenderCacher().RecoverFromBlocks(types.MakeSigner(bc.chainConfig, chain[0].Number(), chain[0].Time()), chain)
```
见：
[[transaction_signing.go]]
[[Sender_cacher.go]]

另起若干线程，开始计算transaction 的signer 地址。


## verify header
```go
for i, block := range chain {
    headers[i] = block.Header() // chain 是入参，要被插入的链
}
abort, results := bc.engine.VerifyHeaders(bc, headers)
defer close(abort)
```

详见：
[[concensus]]

## insert iterator
在 `blockchain_insert.go` 中。

```go
// next returns the next block in the iterator, along with any potential validation
// error for that block. When the end is reached, it will return (nil, nil).
func (it *insertIterator) next() (*types.Block, error) {
	// If we reached the end of the chain, abort
	if it.index+1 >= len(it.chain) {
		it.index = len(it.chain)
		return nil, nil
	}
	// Advance the iterator and wait for verification result if not yet done
	it.index++
	if len(it.errors) <= it.index {
		it.errors = append(it.errors, <-it.results)
	}
	if it.errors[it.index] != nil {
		return it.chain[it.index], it.errors[it.index]
	}
	// Block header valid, run body validation and return
	return it.chain[it.index], it.validator.ValidateBody(it.chain[it.index])
}
```
有两个作用：得到下一个chain，以及检查是否valid。
关于validity，见：[[block_validator.go]]
至于next 本身的逻辑，只是 index++ 而已。

## 跳过已经处理的
```go
  

	// Left-trim all the known blocks that don't need to build snapshot
	if bc.skipBlock(err, it) {
		// First block (and state) is known
		//   1. We did a roll-back, and should now do a re-import
		//   2. The block is stored as a sidechain, and is lying about it's stateroot, and passes a stateroot
		//      from the canonical chain, which has not been verified.
		// Skip all known blocks that are behind us.
		current := bc.CurrentBlock()
		for block != nil && bc.skipBlock(err, it) {
			if block.NumberU64() > current.Number.Uint64() || bc.GetCanonicalHash(block.NumberU64()) != block.Hash() {
				break
			}
			log.Debug("Ignoring already known block", "number", block.Number(), "hash", block.Hash())
			stats.ignored++

			block, err = it.next()
		}
		// The remaining blocks are still known blocks, the only scenario here is:
		// During the snap sync, the pivot point is already submitted but rollback
		// happens. Then node resets the head full block to a lower height via `rollback`
		// and leaves a few known blocks in the database.
		//
		// When node runs a snap sync again, it can re-import a batch of known blocks via
		// `insertChain` while a part of them have higher total difficulty than current
		// head full block(new pivot point).
		for block != nil && bc.skipBlock(err, it) {
			log.Debug("Writing previously known block", "number", block.Number(), "hash", block.Hash())
			if err := bc.writeKnownBlock(block); err != nil {
				return nil, it.index, err
			}
			lastCanon = block

			block, err = it.next()
		}
		// Falls through to the block import
	}
```

上面的代码还可以细分为两部分：
跳过已知部分：
```go
for block != nil && bc.skipBlock(err, it) { ... }
```

处理已知但是需要重写的区块：
```go
if err := bc.writeKnownBlock(block); err != nil { ... }
```

## 区块逐个执行和插入
```go
for ; block != nil && err == nil || errors.Is(err, ErrKnownBlock); block, err = it.next() {
	...
	statedb, err := state.New(parent.Root, bc.statedb)
	...
	res, err := bc.processBlock(block, statedb, start, setHead)
	...
}
```
这里就是核心逻辑了，之后该函数结束，返回witness 和成功插入数。


# 其他辅助模块
## witness

## snapshot
它被用于跳过处理过的区块，用于快速查询world state Trie 减少对底层 Trie 的频繁读取。

- 如果我们能通过 snapshot 快速访问状态，就可以跳过繁重的状态重建过程。
- 如果 snapshot 缺失，就需要重新执行区块以构建状态。

详见 [[snapshot]] 。