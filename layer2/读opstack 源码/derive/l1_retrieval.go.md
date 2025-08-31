# 作用
主要提供了下面的函数

```go
// NextData does an action in the L1 Retrieval stage
// If there is data, it pushes it to the next stage.
// If there is no more data open ourselves if we are closed or close ourselves if we are open
func (l1r *L1Retrieval) NextData(ctx context.Context) ([]byte, error) {
    if l1r.datas == nil {
        next, err := l1r.prev.NextL1Block(ctx)

        if l1r.datas, err = l1r.dataSrc.OpenData(ctx, next, l1r.prev.SystemConfig().BatcherAddr); err != nil {
            return nil, fmt.Errorf("failed to open data source: %w", err)
        }
    }

    l1r.log.Debug("fetching next piece of data")
    data, err := l1r.datas.Next(ctx)
    return data, nil
}
```
其中，`l1r.prev.NextL1Block(ctx)`  就是简单拿来了l1_traversal 的光标处的block。
实现这个功能的核心是data source。见op-node/rollup/derive/data_source.go，具体实现有下面3种：
- op-node/rollup/derive/calldata_source.go
- op-node/rollup/derive/blob_data_source.go
- op-node/rollup/derive/altda_data_source.go

这里涉及到了Data availablity 的概念。

从DA 层取数据，使用的是 `L1TransactionFetcher`，通过网络下载交易数据，见`EthClient`

## data availability

### 1. `calldata_source.go`

- **最基础的实现方式**
- 从 L1 区块的交易中提取 calldata，解析出 batcher 提交的 L2 数据
- 所有 Rollup 都支持的通用机制

### 2. `blob_data_source.go`

- EIP-4844 引入的新机制
- 使用 blob（带证明的低成本大数据块）存储 L2 批次数据
- 更便宜、更高效，但需要 L1 blob 提交支持
- 通常和 `calldata_source` 搭配使用，作为替代或优先方式

### 3. `altda_data_source.go`

- Optimism 专属的扩展
- 允许 batcher 将数据存储到外部 DA 层（如 Celestia、EigenDA）
- 在 L1 上只存储 commitment（摘要）

### batcher
Batcher 是 Optimism 中将 L2 交易打包并提交到 L1 的组件或角色。它从L2网络中收集交易，将这些数据提交到L1 的DA 层。
于是流程大概是这样：
用户交易 → Sequencer L2 执行 → Batcher 收集 → 提交到 L1 → L2 derive pipeline 再从 L1 读取回来

相关阅读：[layers](https://docs.optimism.io/stack/components) ，包括DA 层的概念。

关于batcher，详见：[[op-batcher]]

