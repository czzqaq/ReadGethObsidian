#todo 
# 概览

定义了两个接口，分别是：
## Database
```go
// Database wraps access to tries and contract code.
type Database interface {
	// Reader returns a state reader associated with the specified state root.
	Reader(root common.Hash) (Reader, error)

	// OpenTrie opens the main account trie.
	OpenTrie(root common.Hash) (Trie, error)

	// OpenStorageTrie opens the storage trie of an account.
	OpenStorageTrie(stateRoot common.Hash, address common.Address, root common.Hash, trie Trie) (Trie, error)

	// PointCache returns the cache holding points used in verkle tree key computation
	PointCache() *utils.PointCache

	// TrieDB returns the underlying trie database for managing trie nodes.
	TrieDB() *triedb.Database

	// Snapshot returns the underlying state snapshot.
	Snapshot() *snapshot.Tree
}

```

Database, 底层硬盘读写、数据获取、cache等内容相关的抽象。
仅有一处实现 `CachingDb`，也在这个文件夹里。

## trie
一个针对 world state 的trie。详见 [[trie]]

# 重点接口
这里主要简单写写接口在 `core/state` 下的作用。
## Commit

```go
type Trie interface {
    /// ...
    Commit(collectLeaf bool) (common.Hash, *trienode.NodeSet)
}
```

## UpdateStorage,DeleteStorage


## hash()

