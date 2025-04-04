# eth Overview
 
```ccard
type: folder_brief_live
```
 #目录 

## safeblock
```go
func (api *ConsensusAPI) forkchoiceUpdated(update engine.ForkchoiceStateV1, payloadAttributes *engine.PayloadAttributes, payloadVersion engine.PayloadVersion, payloadWitness bool) (engine.ForkChoiceResponse, error)
```
涉及到了safeblock 的更新。

## snapsync
downloader 中，使用了 blockchain.CurrentSnapBlock()
```go
	switch d.getMode() {
	case ethconfig.FullSync:
		chainHead = d.blockchain.CurrentBlock()
	case ethconfig.SnapSync:
		chainHead = d.blockchain.CurrentSnapBlock()
	default:
		panic("unknown sync mode")
	}
```
### 🔵 FullSync（完整同步）

- 从创世区块开始读取区块。
- 验证每个区块中的交易和状态转换。
- 重放每一笔交易，构建状态树。
- 非常安全，但非常慢。

### 🟣 SnapSync（快照同步）

- 直接从其他节点下载某个区块高度的状态快照（state trie snapshot）。
- 然后从该区块向后验证历史区块，向前同步新区块。
- 跳过了重放所有历史交易这一步，因此显著加快了同步速度。