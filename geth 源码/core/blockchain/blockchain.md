# blockchain Overview
 
```ccard
type: folder_brief_live
```
 


# 阅读指南
从 [[blockchain.go-insertChain]] 开始，剩余两个文件 [[blockchain.go-writeBlock]] 和 [[blockchain.go-chain head]]，是对insertChain 中涉及调用的补充。
insert chain 中，涉及到了链结构的调整（此文件夹下内容），rawdb 的修改（不重点关注，理解rawdb 是一个硬盘中的数据库，新建链需要从rawdb 中恢复，以及frozen 的概念([[blockchain.go-chain head#frozen]])即可），交易的执行（见[[state_transition.go]]），statedb 的修改和commit（见[[StateDB.go]]）。

很难说我读完了整个block chain 中的内容，但是至少 `blockchain.go` 中少有遗漏。

