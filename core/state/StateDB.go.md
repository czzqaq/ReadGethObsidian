整个world state 。原理见：[[4.1 world state]]

是stateObject 的集合。类似于：
$$
stateObject = \sigma(a) = statedb[a]
$$
（不是等于的关系，毕竟程序里有很多其他内容）


# 代码
## getStateObject
```go
// getStateObject retrieves a state object given by the address, returning nil if
// the object is not found or was deleted in this execution context.
func (s *StateDB) getStateObject(addr common.Address) *stateObject {
	// Prefer live objects if any is available
	if obj := s.stateObjects[addr]; obj != nil {
		return obj
	}
	// Short circuit if the account is already destructed in this block.
	if _, ok := s.stateObjectsDestruct[addr]; ok {
		return nil
	}
	s.AccountLoaded++

	start := time.Now()
	acct, err := s.reader.Account(addr)
	if err != nil {
		s.setError(fmt.Errorf("getStateObject (%x) error: %w", addr.Bytes(), err))
		return nil
	}
	s.AccountReads += time.Since(start)

	// Short circuit if the account is not found
	if acct == nil {
		return nil
	}
	// Schedule the resolved account for prefetching if it's enabled.
	if s.prefetcher != nil {
		if err = s.prefetcher.prefetch(common.Hash{}, s.originalRoot, common.Address{}, []common.Address{addr}, nil, true); err != nil {
			log.Error("Failed to prefetch account", "addr", addr, "err", err)
		}
	}
	// Insert into the live set
	obj := newObject(s, addr, acct)
	s.setStateObject(obj)
	s.AccountLoaded++
	return obj
}
```

## getState
详见：[[state_object.go#getState]]

## commit
```go
// Commit writes the state mutations into the configured data stores.
//
// Once the state is committed, tries cached in stateDB (including account
// trie, storage tries) will no longer be functional. A new state instance
// must be created with new root and updated database for accessing post-
// commit states.
//
// The associated block number of the state transition is also provided
// for more chain context.
//
// noStorageWiping is a flag indicating whether storage wiping is permitted.
// Since self-destruction was deprecated with the Cancun fork and there are
// no empty accounts left that could be deleted by EIP-158, storage wiping
// should not occur.
func (s *StateDB) Commit(block uint64, deleteEmptyObjects bool, noStorageWiping bool) (common.Hash, error) {
	ret, err := s.commitAndFlush(block, deleteEmptyObjects, noStorageWiping)
	if err != nil {
		return common.Hash{}, err
	}
	return ret.root, nil
}
```

针对stateDB 的操作，效果是：
1. 使得cache 都不再工作。
2. 创建新的state 状态，包括新的root，包括储存commit 后的state。

### 调用例：
1. 用户输入 `geth import` 
2. 调用 cmd/geth/chaincmd.go:importChain 
3. 调用`cmd/utils/cmd.go:ImportChain`
4. 调用`core/blockchain.go:insertChain`
5. 调用 `core/blockchain.go:processBlock` 这个函数验证了import 的block，并且把它的state 写入到相关database 。
6. `core/blockchain.go:writeBlockWithState` 写入rawdb 和 statedb 的部分
7. `core/state/statedb.go:Commit

### 调用栈

- StateDB.Commit
    - StateDB.commitAndFlush
        - **StateDB.commit**
            - **s.IntermediateRoot 处理pending changes**。见：[[#IntermediateRoot]]
            - 处理账号删除。
            - **更新s.trie**。见 [[database.go#Commit]]
            - 对于每个`s.mutations map[common.Address]*mutation` 中的 account，**提交state_object 上的更新**。`s.stateObjects[addr].commit()` 见：[[state_object.go#commit 和不同cache]]
        - 如果 s.db.TrieDB() 中有数据（dirty contract code)，且更新的状态中包含代码（`stateUpdate.codes`) ，就写入rawdb
        - 如果状态有更新（将要写入的stateroot 发生变化），更新s.db.Snapshot()。更新s.db.TrieDB()

### IntermediateRoot
#### 作用
计算当前的state trie 的root hash。

#### 调用栈
- s.IntermediateRoot
    - **s.Finalise** 见：[[#Finalize]]
    - 停止 s.prefetcher。似乎是一个加载 state trie 的工具。
    -  对于每个`s.mutations` 中的account，**`obj.updateRoot()`** ，见[[state_object.go#updateRoot]]
    - 对于上述每个obj, 调用 s.witness.AddState(obj.trie.Witness())
    - 更多的关于s.witness 的处理。

### Finalize
