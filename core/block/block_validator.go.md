
#archived #概念 

# ValidateBody

## 调用时机

这部分内容对应了 [[4.4.2 Holistic Validity]]

[[blockchain.go-insertChain#insert iterator]] 中调用，在process block之前就做 validate 检查了。

blockbody 包括了(摘自[[4.4.0 block]]）：
>1. **交易列表（`BT`）**: 区块中的所有交易信息。
>2. **已弃用的叔块头数组（`BU`）**: 该字段已无实际用途，内容为空数组。
>3. **提款列表（`BW`）**: 由共识层推送的提款操作集合。

## 代码
### transaction
包括两部分：
一、检查哈希值是否和header 中的 Transaction Root Hash(H_t) 匹配
```go
    if hash := types.DeriveSha(block.Transactions(), trie.NewStackTrie(nil)); hash != header.TxHash {
        return fmt.Errorf("transaction root hash mismatch (header value %x, calculated %x)", header.TxHash, hash)
    }
```
二、处理blob（cancun fork 之后）都看不太懂，之后研究吧 （EIP-4844)
```go
var blobs int
    for i, tx := range block.Transactions() {
        // Count the number of blobs to validate against the header's blobGasUsed
        blobs += len(tx.BlobHashes()
        // If the tx is a blob tx, it must NOT have a sidecar attached to be valid in a block.
        if tx.BlobTxSidecar() != nil {
            return fmt.Errorf("unexpected blob sidecar in transaction at index %d", i)
        }
    }

    // Check blob gas usage.
    if header.BlobGasUsed != nil {
        if want := *header.BlobGasUsed / params.BlobTxBlobGasPerBlob; uint64(blobs) != want { // div because the header is surely good vs the body might be bloated
            return fmt.Errorf("blob gas used mismatch (header %v, calculated %v)", *header.BlobGasUsed, blobs*params.BlobTxBlobGasPerBlob)
        }
    } else {
        if blobs > 0 {
            return errors.New("data blobs present in block body")
        }
    }
```

### uncle block
```go
    if err := v.bc.engine.VerifyUncles(v.bc, block); err != nil {
        return err
    }
    if hash := types.CalcUncleHash(block.Uncles()); hash != header.UncleHash {
        return fmt.Errorf("uncle root hash mismatch (header value %x, calculated %x)", header.UncleHash, hash)
    }
```

也是要让uncle 的哈希值跟Head 中的 H_o，Uncle block hash 对上。因为已经弃用，就不研究了。

### withdraw

```go

	// Withdrawals are present after the Shanghai fork.
	if header.WithdrawalsHash != nil {
		// Withdrawals list must be present in body after Shanghai.
		if block.Withdrawals() == nil {
			return errors.New("missing withdrawals in block body")
		}
		if hash := types.DeriveSha(block.Withdrawals(), trie.NewStackTrie(nil)); hash != *header.WithdrawalsHash {
			return fmt.Errorf("withdrawals root hash mismatch (header value %x, calculated %x)", *header.WithdrawalsHash, hash)
		}
	} else if block.Withdrawals() != nil {
		// Withdrawals are not allowed prior to Shanghai fork
		return errors.New("withdrawals present in block body")
	}
```

1. 如果header 中有withdraw 字段，那么body 上就应该带着withdraw；如果没有，body 就也没有
2. withdraw 的哈希值和header 中字段 $H_w$ withdraw 的root hash 一致。

### 在链中
不能已经存在于链
```go
    if v.bc.HasBlockAndState(block.Hash(), block.NumberU64()) {
        return ErrKnownBlock
    }
```

父节点必须存在于链
```go
	// Ancestor block must be known.
	if !v.bc.HasBlockAndState(block.ParentHash(), block.NumberU64()-1) {
		if !v.bc.HasBlock(block.ParentHash(), block.NumberU64()-1) {
			return consensus.ErrUnknownAncestor
		}
		return consensus.ErrPrunedAncestor
	}
```

只有这种block，可以被插入。

# ValidateState
## 说明
在[[blockchain.go-insertChain#validator]]  中被使用，是处理完交易后调用的。包括了对Receipt、Request、stateDB 的检查。

把交易的结果 `ProcessResult` 和block 中本来就有的储存状态比较，从而对区块的合法性校验。

### Receipt 的来源
对于 ProcessResult 的来源，见：[[blockchain.go-insertChain#返回 ProcessResult]] ，其实就是 [[4.4.1 Transaction Receipt]] 的部分 (加上Prague 升级产生的requests) 

### stateDB 的来源

1. [[blockchain.go-insertChain]] 中新建了一个stateDB，使用的worldstate 是链中保存的，插入这个block前的状态。
2. 在[[blockchain.go-insertChain#ApplyTransactionWithEVM]]   被拷贝进evm，随着[[state_transition.go]] 的过程被修改，最终被finalize。

它就是交易执行完成后的新状态。

### block 的来源

而区块上的储存状态，例如 `header.GasUsed`，是本身输入的区块中自带的。可以认为整个block insert 的过程中，入参block 基本是const 的，这个过程虽然execute 了一遍Transaction，但并没有修改block 内的状态。

## 代码
### gas used 相同
```go
    if block.GasUsed() != res.GasUsed {
        return fmt.Errorf("invalid gas used (remote: %d local: %d)", block.GasUsed(), res.GasUsed)
    }
```
对应 [[12 Block Finalisation#12.2 交易验证（**Transaction Validation**）]]
### log bloom
header 中的 $H_b$ **logsBloom** 字段是Receipt 中全部logs 的merge：
```go
    rbloom := types.MergeBloom(res.Receipts)
    if rbloom != header.Bloom {
        return fmt.Errorf("invalid bloom (remote: %x  local: %x)", header.Bloom, rbloom)
    }
```

```go
func MergeBloom(receipts Receipts) Bloom {
    var bin Bloom
    for _, receipt := range receipts {
        if len(receipt.Logs) != 0 {
            bl := receipt.Bloom.Bytes()
            for i := range bin {
                bin[i] |= bl[i]
            }
        }
    }
    return bin
}
```

### Receipt Trie
检查区块头中的$H_e$ 字段，即Receipt Hash Root。
```go
    receiptSha := types.DeriveSha(res.Receipts, trie.NewStackTrie(nil))
    if receiptSha != header.ReceiptHash {
        return fmt.Errorf("invalid receipt root hash (remote: %x local: %x)", header.ReceiptHash, receiptSha)
    }
```

### request
在 prague fork 之后的区块，可能带Request 字段
```go
	if header.RequestsHash != nil {
		reqhash := types.CalcRequestsHash(res.Requests)
		if reqhash != *header.RequestsHash {
			return fmt.Errorf("invalid requests hash (remote: %x local: %x)", *header.RequestsHash, reqhash)
		}
	} else if res.Requests != nil {
		return errors.New("block has requests before prague fork")
	}
```

### 对StateDB 的检查
```go
    if root := statedb.IntermediateRoot(v.config.IsEIP158(header.Number)); header.Root != root {
        return fmt.Errorf("invalid merkle root (remote: %x local: %x) dberr: %w", header.Root, root, statedb.Error())
    }
```
其中，[[StateDB.go#IntermediateRoot]] 把StateDB finalize，然后计算了state Root Hash，这里就是要state trie 的根哈希。
实际上对应了[[12 Block Finalisation#12.3 状态验证（**State Validation**）]]

# CalcGasLimit

## 原理

参考 [[4.4.4 Block header validility#**区块 Gas 限制（$H_l$）的约束条件**]] 中的原理。
虽然叫 CalcGasLimit，但其实这个函数的作用是用限制的最大最小值箍一下 `desiredLimit`. 这个值是矿工设定的，见：[[miner]]

付代码：
```go
// CalcGasLimit computes the gas limit of the next block after parent. It aims
// to keep the baseline gas close to the provided target, and increase it towards
// the target if the baseline gas is lower.
func CalcGasLimit(parentGasLimit, desiredLimit uint64) uint64 {
	delta := parentGasLimit/params.GasLimitBoundDivisor - 1
	limit := parentGasLimit
	if desiredLimit < params.MinGasLimit {
		desiredLimit = params.MinGasLimit
	}
	// If we're outside our allowed gas range, we try to hone towards them
	if limit < desiredLimit {
		limit = parentGasLimit + delta
		if limit > desiredLimit {
			limit = desiredLimit
		}
		return limit
	}
	if limit > desiredLimit {
		limit = parentGasLimit - delta
		if limit < desiredLimit {
			limit = desiredLimit
		}
	}
	return limit
}

```

