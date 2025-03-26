
## Commit

```go
type Trie interface {
    /// ...
    Commit(collectLeaf bool) (common.Hash, *trienode.NodeSet)
}
```
