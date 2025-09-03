# 作用

主要是定义了下面的函数。执行过程：
L1BlockRefByNumber，得到下一个Number+1 的block，
检查下一个block 的parent hash 是不是就是当前的block，也就是确定没有出现reorg。
之后FetchReceipts

```go
func (l1t *L1Traversal) AdvanceL1Block(ctx context.Context) error {
    origin := l1t.block
    nextL1Origin, err := l1t.l1Blocks.L1BlockRefByNumber(ctx, origin.Number+1)
    if l1t.block.Hash != nextL1Origin.ParentHash {
        return NewResetError(fmt.Errorf("detected L1 reorg from %s to %s with conflicting parent %s", l1t.block, nextL1Origin, nextL1Origin.ParentID()))
    }

    _, receipts, err := l1t.l1Blocks.FetchReceipts(ctx, nextL1Origin.Hash)

    l1t.block = nextL1Origin
    l1t.done = false

    return nil
}
```
## L1BlockRefByNumberFetcher
### 功能实现
```go
type L1BlockRefByNumberFetcher interface {
    L1BlockRefByNumber(context.Context, uint64) (eth.L1BlockRef, error)
    FetchReceipts(ctx context.Context, blockHash common.Hash) (eth.BlockInfo, types.Receipts, error)
}```
其核心功能的实现是由这个Fetcher 实现的。其实现见 op-program/client/l1/client.go，不过归根结底，它是一种Oracle，见：
op-preimage/Oracle.go

其中，Oracle ：
>Oracle 是一个Preimage的服务提供者，客户端通过 key 查询真实数据（value）

注意这个不是chainlink 提供的那个跨链服务的Oracle。preimage 则是hash = f(x), 则x 是hash 的preimage

而如果再继续往下，具体的KV 查询是如何实现的呢？通过Kvstore，差不多就是抽象了的键值对数据库。来源有:
- file op-program/host/kvstore/file.go
- memory op-program/host/kvstore/mem.go
- pebble  op-program/host/kvstore/pebble.go

以file 为例
```go
func (d *fileKV) Get(k common.Hash) ([]byte, error) {
    d.RLock()
    defer d.RUnlock()

    f, err := os.OpenFile(d.pathKey(k), os.O_RDONLY, filePermission)
    if err != nil {
        return nil, fmt.Errorf("failed to open pre-image file %s: %w", k, err)
    }
    defer f.Close() // fine to ignore closing error here
    
    dat, err := io.ReadAll(f)
    return hex.DecodeString(string(dat))
}
```

### 架构
#### oracle
在 op-program/host/host.go: RunProgram() 的函数中，我们启用了一个preimageServer, 并把它绑定在新创建的Oracle 上。
```go
pClient := preimage.NewOracleClient(preimageServer.PreimageClientRW())
```
Oracle 调用Get 获得preimage 的具体过程，由 preimageServer 完成。
这个 preimageServer 就是一个Kvstore，在runprogram 时由配置指定是哪一个。

```go
    if err = p.executor.RunProgram(ctx, p, header.NumberU64()+1, agreedOutput, chainID, hostcommon.NewL2KeyValueStore(p.kvStore)); err != nil {
        return err
    }
```

#### KV store
op-program/host/kvstore 下做了关于KVstore 的具体实现。其抽象接口：

```go
// KV is a Key-Value store interface for pre-image data.
type KV interface {
    // Put puts the pre-image value v in the key-value store with key k.
    // KV store implementations may return additional errors specific to the KV storage.
    Put(k common.Hash, v []byte) error
    // Get retrieves the pre-image with key k from the key-value store.
    // It returns ErrNotFound when the pre-image cannot be found.
    // KV store implementations may return additional errors specific to the KV storage.
    Get(k common.Hash) ([]byte, error)
    // Closes the KV store.
    Close() error
}
```

