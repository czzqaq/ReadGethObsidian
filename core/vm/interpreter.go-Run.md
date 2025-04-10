# 流程解析
## 前期准备
### readOnly
入参有字段 readOnly
```go
func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error)
```
readOnly 指这个指令不应该影响 合约地址的状态（包括储存等）。对应 I_w 字段，是solidity的view 和 pure 字段。

```go
    if readOnly && !in.readOnly {
        in.readOnly = true
        defer func() { in.readOnly = false }()
    }
```


### 初始化上下文
```go
mem := NewMemory()
stack := newstack()
callContext := &ScopeContext{...}
```
memory 见： [[memory]]
stack 就是一个自己实现的stack，支持[[jump_table.go#SWAP 系列 (0x90 ~ 0x9F)]]]，
contract 是入参，包含了调用者地址、合约地址、合约代码等。

### 代码运行相关
- pc: 从0开始，program counter，code 是一串二进制，用来从code 上取执行的内容，比如：`op = contract.GetOp(pc)`，`operation.execute(&pc, in, callContext)` 
- constract.gas: 并非是指contract 地址下的balance，单纯是对gas limit 扣除各项费用后剩余gas 的统计。

## 主循环
```go
for {
        op = contract.GetOp(pc)
        
        ... 
        
        res, err = operation.execute(&pc, in, callContext)
        if err != nil {
            break
        }
        pc++
}
```

从一个for 开始，循环直到遇到错误、或者遇到表示终止的命令，比如STOP等，所有返回err 的execute 都会终止循环。例如：
```go
func opRevert(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	...
	return ret, ErrExecutionReverted
}
```


每次迭代代表执行一个 EVM 指令。

### 获取指令

```go
op = contract.GetOp(pc)
operation := in.table[op]
```

- 获取当前指令和对应的操作函数。

### 堆栈检查
```go
if sLen := stack.len(); sLen < operation.minStack { ... }
```

- 检查堆栈是否满足操作要求（防止下溢/上溢）。

### 基础 Gas 消耗

```go
cost = operation.constantGas
if contract.Gas < cost {
	return nil, ErrOutOfGas
}
contract.Gas -= cost
```

- 扣除操作的基础 Gas。

### 动态 Gas 与内存扩展

```go
dynamicCost, err = operation.dynamicGas(in.evm, contract, stack, mem, memorySize)
if operation.dynamicGas != nil {
	...
}
contract.Gas -= dynamicCost
```

- 如果操作需要动态 Gas（如 `CALL`, `SSTORE`），则计算并扣除。

### 内存
```go
if operation.memorySize != nil {
    memSize, overflow := operation.memorySize(stack)
    memorySize = ceil(memSize / 32) * 32
    }
    
if memorySize > 0 {
	mem.Resize(memorySize)
}
```
对于mem 的用法，详见memory 。

### 执行操作

```go
res, err = operation.execute(&pc, in, callContext)
if err != nil {
	break
}
pc++
```

- 执行指令并更新 PC。如果出错，退出循环。


# 总结
## 错误检查

包括了overflow, underflow, gas fee 和 invalid oper 的检查，剩余的错误均是指令特有的。
![[Pasted image 20250410083617.png]]
其中，readonly 的检查也在execute 中，包括了：
[[jump_table.go#LOG 系列 (0xA0 ~ 0xA4)]] [[jump_table.go#CREATE / CALL / RETURN / SELFDESTRUCT 等 (0xF0 ~ 0xFF)]]

