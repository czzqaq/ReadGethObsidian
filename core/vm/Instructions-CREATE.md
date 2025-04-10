# 说明
本章将会分析CREATE和CREATE2 的内容，两个指令一起分析。我会先分析create，再基于不同，提一下create2.
类似CALL，先分析evm.Create，看看它是怎么具体执行的，对应主要对应[[7. Contract Creation]] 。再分析 Instructions.go-opCreate，主要对应 [[9. Execution Model]] 
