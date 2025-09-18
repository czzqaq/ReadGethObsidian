# op-batcher Overview
 
```ccard
type: folder_brief_live
```
 

# 概念
## 在架构中
https://specs.optimism.io/protocol/batcher.html

根据这段话，我们知道：
1. batch 的数据依赖 [[sequencing]]，它是把L2 的区块链的维持者。
2. 提交的目的地是 L1 DA 层
3. 被提交的部分是 unsafe block。（这很好理解，因为 [safe block](https://specs.optimism.io/glossary.html?highlight=unsafe#safe-l2-block) 的定义就是可以从L1 derive 的block，已经在 L1 上了) 

另外，注意 batcher 和 [[derive]] 的关系，derive pipeline 从 DA 层中读数据，即依赖batch 的输出结果。

sequencer -> batcher -> L1 DA -> derivation 的关系如下图所示。
![[architecture.png]]


## 内部处理
  
![[Pasted image 20250918170118.png]]
其中，可以看到 4个loop，见：[[driver.go]]
