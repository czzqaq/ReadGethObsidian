
功能很复杂，其实就是快速查询world state 用的。

```go
type Snapshot interface {
	Root() common.Hash
	Account(hash common.Hash) (*types.SlimAccount, error)
	AccountRLP(hash common.Hash) ([]byte, error)
	Storage(accountHash, storageHash common.Hash) ([]byte, error)
}
```