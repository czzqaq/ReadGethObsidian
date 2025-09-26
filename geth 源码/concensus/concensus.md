#目录 #archived 

# concensus Overview
 
```ccard
type: folder_brief_live
```
todo:
```go
if p.config.DAOForkSupport && p.config.DAOForkBlock != nil && p.config.DAOForkBlock.Cmp(block.Number()) == 0 { misc.ApplyDAOHardFork(statedb) }
```

# 文件结构
consensus/
 ┣ beacon/
 ┣ clique/
 ┣ ethash/
 ┣ misc/
 ┃ ┣ eip1559/
 ┃ ┣ eip4844/
 ┃ ┣ dao.go
 ┃ ┗ gaslimit.go
 ┣ consensus.go
 ┗ errors.go

其中：
- **beacon** 是对于 POS 共识算法的一个极其简单的实现，geth 并不负责beacon chain，于是它只有300行，而且非常不全。
- **clique** PoA（Proof of Authority）相关内容。"`geth` comes with a native Proof-of-Authority protocol called `Clique`" 。这部分的内容相对独立，只和POA 相关，因此我暂时不阅读它。
- **ethash**  Ethash 是Ethereum 1.0 计划的PoW 共识算法。这部分和 POW 的 ethreum 1.0有关，我暂时不阅读它。
- **misc** 详见[[misc 目录-dao]]
- `consensus.go`  仅仅规定了一个接口，声明了共识算法应当有哪些行为。`type Engine interface`。这个接口分别被 `beacon\`, `clique\`, `ethash\` 实现。

# 阅读指南
这部分的代码量很少，本身确实就很短，再考虑到它实现了POW，POA，POS 三种共识算法，就更短了。确实共识也不是 geth 的主业。我不做彻底的拆解，而仅仅是遇到一个功能，就增加一点功能的分析。
在 [[POS 共识]]，对具体的POS 共识算法做了说明。