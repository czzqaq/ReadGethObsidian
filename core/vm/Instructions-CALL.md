# 预先知识

[[8. Message Call]] 给出了针对CALL 这个指令的输出的说明，它主要是对statedb 操作。
[[9. Execution Model]] 给了对指令调用检查的部分。以及执行流程


# evm.Call
它被interpreter.opCall [[#opCall]] 所调用，是执行的主体。

## 检查
### 调用栈深度
```go
if evm.depth > int(params.CallCreateDepth) {
	return nil, gas, ErrDepth
}
```

调用深度检查。只有CALL 类的函数和CREATE 有这个检查，在一次 CALL 的调用中，每迭代进入一次 [[interpreter.go-Run]]，就栈深度+1

### tx.value 检查
```go
    if !value.IsZero() && !evm.Context.CanTransfer(evm.StateDB, caller, value) {
        return nil, gas, ErrInsufficientBalance
    }
```
只有CALL，CREATE，CREATE2 会存在transfer，它对应着转账：

$$
\sigma_1[r]_b \equiv \sigma[r]_b + v \quad \land \quad \sigma_1[s]_b \equiv \sigma[s]_b - v \quad \text{如果 } s \ne r
$$
## precompile
```go
p, isPrecompile := evm.precompile(addr)
...
if isPrecompile {
        ret, gas, err = RunPrecompiledContract(p, input, gas, evm.Config.Tracer)
    }
```
对于precompile 相关的调用，见：[[appendix.E Precompiled Contracts]]
### RunPrecompiledContract
```go
func RunPrecompiledContract(p PrecompiledContract, input []byte, suppliedGas uint64, logger *tracing.Hooks) (ret []byte, remainingGas uint64, err error) {
    gasCost := p.RequiredGas(input)
    if suppliedGas < gasCost {
        return nil, 0, ErrOutOfGas
    }
    suppliedGas -= gasCost
    output, err := p.Run(input)
    return output, suppliedGas, err
}
```
对于每个Precompiled contract，其运行都是使用直接编好的逻辑，难怪预编译合约使用的gas fee 更少。举一个例子：
```go
func (c *sha256hash) RequiredGas(input []byte) uint64 {
    return uint64(len(input)+31)/32*params.Sha256PerWordGas + params.Sha256BaseGas
}
func (c *sha256hash) Run(input []byte) ([]byte, error) {
    h := sha256.Sum256(input)
    return h[:], nil
}
```

## 转账
### 空地址处理

#### EIP-4762 方案
生成访问证明（witness），并为其付 gas 费。
```go
        if !isPrecompile && evm.chainRules.IsEIP4762 && !isSystemCall(caller) {
            // add proof of absence to witness
            wgas := evm.AccessEvents.AddAccount(addr, false)
            if gas < wgas {
                evm.StateDB.RevertToSnapshot(snapshot)
                return nil, 0, ErrOutOfGas
            }
            gas -= wgas
}
```
#### EIP-158 方案
黄皮书上的方案。如果没有转账，就直接什么都不做。即空地址和不存在地址等同。
```go
  if !isPrecompile && evm.chainRules.IsEIP158 && value.IsZero() {
            // Calling a non-existing account, don't do anything.
            return nil, gas, nil
}
```

#### 默认处理
否则，创建这个账户（在world state 中增加它），之后会改写它的value，其余内容为空。因为实际上，根据[[8. Message Call#转账与账户状态初步更新]]，无论是不是空账户，其实message call 的初步更新步骤都是仅仅包括了转账，即地址value 的更新。
```go


        evm.StateDB.CreateAccount(addr)
```

### transfer
```go
evm.Context.Transfer(evm.StateDB, caller, addr, value)
```
其中：
```go
func Transfer(db vm.StateDB, sender, recipient common.Address, amount *uint256.Int) {
    db.SubBalance(sender, amount, tracing.BalanceChangeTransfer)
    db.AddBalance(recipient, amount, tracing.BalanceChangeTransfer)
}
```

## 运行

```go
        code := evm.resolveCode(addr)
        if len(code) == 0 {
            ret, err = nil, nil // gas is unchanged
        } else {
            // The contract is a scoped environment for this execution context only.
            contract := NewContract(caller, addr, value, gas, evm.jumpDests)
            contract.IsSystemCall = isSystemCall(caller)
            contract.SetCallCode(evm.resolveCodeHash(addr), code)
            ret, err = evm.interpreter.Run(contract, input, false)
            gas = contract.Gas
        }
```
### 变量准备
#### resolveCode
只要不是 Prague hard fork(EIP-7702)，就直接：`return evm.StateDB.GetCode(addr)`

#### NewContract
对于contract 的操作都是为了给它赋值。

### Run
详见：[[interpreter.go-Run]]
这个调用开始解析code，然后产生运行opcode 对应的运行结果。注意到每个contract 都是一个独立的程序的感觉，当然contract 确实也可以再调用contract，不过它和函数的jump 在底层完全不同。


## 错误处理
```go
	if err != nil {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != ErrExecutionReverted {
			gas = 0
		}
	}
```
对于gas = 0 见：[[8. Message Call#剩余 gas：]]
其中，revert 是指在solidity中：

```solidity
require(x > 0, "Invalid x");
```
的情况，其他情况都是因为合约代码本身出错，这种情况下，`gaslimit` 的gas 都会被消耗。

此外，会`RevertToSnapshot`，详见[[journal.go#revertToSnapshot]]，也就是把上次commit 之前到现在的所有dirtied 修改返回。



# opCall


## 转账的处理
### 只读保护：阻止 value 转账

```go
if interpreter.readOnly && !value.IsZero() {
	return nil, ErrWriteProtection
}
```

如果当前解释器处于只读模式（如 `eth_call`），**不能发送 ETH（即 `value > 0`）**，否则将返回 `ErrWriteProtection` 错误。

---

### 添加 gas stipend

```go
if !value.IsZero() {
	gas += params.CallStipend
}
```

- 如果发送了 ETH（`value > 0`），自动附加 **2300 gas stipend**（用于 fallback/receive 函数执行）。
- 这个 stipend 是 EVM 的特性，保证接收方有最低 gas 可用。

##  发起内部 CALL 调用
### args
根据黄皮书 [[8. Message Call]]

在执行消息调用时，需要以下参数：
- 发送者 $s$
- 交易发起者 $o$
- 接收者 $r$（即将执行代码的账户）
- 代码所在账户 $c$（通常与 $r$ 相同）
- 可用 gas $g$
- 转账金额 $v$
- 执行上下文中的金额 $\tilde{v}$（用于区分 DELEGATECALL）
- 有效 gas 价格 $p$
- 输入数据（字节数组）$d$
- 当前调用栈深度 $e$
- 是否允许状态修改权限 $w$

其中，s,r, g, v 来自栈，w 是入参，就天然有一类readOnly 调用。

### 取参数
```go
stack := scope.Stack
temp := stack.pop()
gas := interpreter.evm.callGasTemp
addr, value, inOffset, inSize, retOffset, retSize := stack.pop(), stack.pop(), stack.pop(), stack.pop(), stack.pop(), stack.pop()
```

|参数位置|含义|
|---|---|
|1|**Gas**（本次调用最多可用的 gas）|
|2|**目标地址**（即 call 的目标地址）|
|3|**发送的 ETH 数量**（`value`）|
|4|**输入数据偏移（memory offset）**|
|5|**输入数据长度**|
|6|**返回数据写入偏移（memory offset）**|
|7|**返回数据写入长度**|

这里 `temp` 是栈顶的 gas，但它没有直接用，而是用 `interpreter.evm.callGasTemp`，这是在外部预设的 gas 值（符合 EIP-150 规则）。

另外，
```go
toAddr := common.Address(addr.Bytes20())
```

### input
```go
args := scope.Memory.GetPtr(inOffset.Uint64(), inSize.Uint64())

```
不同合约，不同函数，有不同的解读input 的方式，使用例：
```go
func runBn256Add(input []byte) ([]byte, error) {
	x, err := newCurvePoint(getData(input, 0, 64))
	if err != nil {
		return nil, err
	}
	y, err := newCurvePoint(getData(input, 64, 64))
	if err != nil {
		return nil, err
	}
	res := new(bn256.G1)
	res.Add(x, y)
	return res.Marshal(), nil
}
```

### Call
详见[[#evm.Call]]
```go
ret, returnGas, err := interpreter.evm.Call(scope.Contract.Address(), toAddr, args, gas, &value)
```
调用 `evm.Call(...)` 发起内部合约调用：

- 来自地址 = 当前合约地址
- 目标地址 = `toAddr`
- 输入数据 = `args`
- Gas = 调整后的可用 gas
- ETH = `value`

返回：

- `ret`: 被调用合约返回的字节数据
- `returnGas`: 未使用的 gas
- `err`: 调用是否失败


### 设置返回值到栈中

```go
if err != nil {
	temp.Clear()
} else {
	temp.SetOne()
}
stack.push(&temp)
```

- CALL 指令的返回值 $z$ 是一个布尔值（`0` 或 `1`）：
    - `0`: 执行失败
    - `1`: 执行成功
- 这个值被压回栈顶。

### 写入返回数据到内存（如果没报错或是 REVERT）

```go
if err == nil || err == ErrExecutionReverted {
	scope.Memory.Set(retOffset.Uint64(), retSize.Uint64(), ret)
}
```

- 如果调用成功，或是 `REVERT`（虽然状态回滚，仍返回 data），则把返回值写入本合约内存。
- 写入位置和长度由 `retOffset`, `retSize` 决定。

## 退还多余 gas

```go
scope.Contract.RefundGas(returnGas, interpreter.evm.Config.Tracer, tracing.GasChangeCallLeftOverRefunded)
```

- 将没用完的 gas 退还给父合约。
- 这个操作可能被 tracer 记录下来。
