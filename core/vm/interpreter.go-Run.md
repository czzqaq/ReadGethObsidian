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

## 主循环
从一个for 开始，直到此循环结束。
每次迭代代表执行一个 EVM 指令（opcode）：


#### a. **获取指令并检查堆栈**

```go
op = contract.GetOp(pc)
operation := in.table[op]
```

- 获取当前指令和对应的操作函数。

```go
if sLen := stack.len(); sLen < operation.minStack { ... }
```

- 检查堆栈是否满足操作要求（防止下溢/上溢）。

---

#### b. **基础 Gas 消耗**

```go
if contract.Gas < cost {
	return nil, ErrOutOfGas
}
contract.Gas -= cost
```

- 扣除操作的基础 Gas。

---

#### c. **动态 Gas 与内存扩展**

```go
if operation.dynamicGas != nil {
	...
}
```

- 如果操作需要动态 Gas（如 `CALL`, `SSTORE`），则计算并扣除。

```go
if memorySize > 0 {
	mem.Resize(memorySize)
}
```

- 扩展内存（如果需要）。

---

#### d. **调试信息（可选）**

```go
if debug {
	in.evm.Config.Tracer.OnOpcode(...)
}
```

- 在执行前记录调试信息。

---

#### e. **执行操作**

```go
res, err = operation.execute(&pc, in, callContext)
if err != nil {
	break
}
pc++
```

- 执行指令并更新 PC。如果出错，退出循环。
