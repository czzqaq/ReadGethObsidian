#archived 

整个world state 。原理见：[[4.1 world state]]

是stateObject 的集合。类似于：
$$
stateObject = \sigma(a) = statedb[a]
$$
（不是等于的关系，毕竟程序里有很多其他内容）


# stateObject 相关代码
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
返回当前的状态。详见：[[state_object.go#getState]]

## commit
针对stateDB 的操作，效果是：
1. 使得cache 都不再工作。
2. 创建新的state 状态，包括新的root，包括储存commit 后的state。
返回新的world state 全局root hash。

### 使用例：
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
            - 清理所有的临时cache和中间状态。
            - 调用 newStateUpdate, 详见 [[stateupdate.go]]
        - 如果 s.db.TrieDB() 中有数据（dirty contract code)，且更新的状态中包含代码（`stateUpdate.codes`) ，就写入rawdb
        - 如果状态有更新（将要写入的stateroot 发生变化），更新s.db.Snapshot()。更新s.db.TrieDB()
	- 返回新的world state hash

### IntermediateRoot
#### 作用
计算当前的state trie 的root hash，更新object 的状态。

#### 调用栈
- s.IntermediateRoot
    - **s.Finalise** 见：[[#Finalize]]
    - 停止 s.prefetcher。似乎是一个加载 state trie 的工具。
    -  对于每个`s.mutations` 中的account，**`obj.updateRoot()`** ，见[[state_object.go#updateRoot]]
    - 对于上述每个obj, 调用 s.witness.AddState(obj.trie.Witness())
    - 更多的关于s.witness 的处理。

### Finalize
#### 作用
对于每个**脏的地址** addr ：关于脏地址的统计，见：[[journal.go]]
obj = s.stateObjects\[addr\]
删除被destruct 或者空的 obj。
对于其他的object，**调用：obj.finialize** . [[state_object.go#finalize]]


# EIP 和补充
## TransientState

见：[[transient_storage.go]]

## prefetcher
见：[[trie_prefetcher.go]]

## access list
包括：
```go
func (s *StateDB) AddAddressToAccessList(addr common.Address)
func (s *StateDB) AddSlotToAccessList(addr common.Address, slot common.Hash)
func (s *StateDB) AddressInAccessList(addr common.Address) bool
func (s *StateDB) SlotInAccessList(addr common.Address, slot common.Hash) (addressPresent bool, slotPresent bool)
```
都是对 `s.accessList` 的处理。详见：[[accesss_list.go]]

# 验证相关
即 [[4.1 world state#存储 Trie 及哈希计算（Storage Trie & Hash Computation）]] 和[[4.1 world state#世界状态哈希计算（World State Hash Computation）]] 中相关的内容

## GetStorageRoot

```go
func (s *StateDB) GetStorageRoot(addr common.Address) common.Hash {
    stateObject := s.getStateObject(addr)
    if stateObject != nil {
        return stateObject.Root()
    }
    return common.Hash{}
}
```

见：[[state_object.go#updateRoot]]

## 世界哈希


关于一次变更带来的trie 改变（也就是多出来的那些p(a))，见：[[state_object.go#commit]]

最终哈希的计算见：[[#commit]]，特别是 [[database.go#Commit]]

