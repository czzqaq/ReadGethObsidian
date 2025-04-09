# 预先知识

[[8. Message Call]] 给出了针对CALL 这个指令的输出的说明，它主要是对statedb 操作。
[[9. Execution Model]] 给了对指令调用检查的部分。以及执行流程



# 代码和流程分析
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
#### EIP158 方案
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



