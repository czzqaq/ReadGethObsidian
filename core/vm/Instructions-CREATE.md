# 说明
本章将会分析CREATE和CREATE2 的内容，两个指令一起分析。我会先分析create，再基于不同，提一下create2.
类似CALL，先分析evm.Create，看看它是怎么具体执行的，对应主要对应[[7. Contract Creation]] 。再分析 Instructions.go-opCreate，主要对应 [[9. Execution Model]] 

# evm.CREATE

## 流程
### 产生合约地址
```go
contractAddr = crypto.CreateAddress(caller, evm.StateDB.GetNonce(caller))
```
详见：[[crypto]]

### 通用检查
有最大调用深度限制。
在create 中也可以转账，如果要转账，则检查是否余额大于value。
这两项和[[Instructions-CALL#检查]] 中的行为一致。


### 设置nonce
```go
    nonce := evm.StateDB.GetNonce(caller)
    if nonce+1 < nonce {
        return nil, common.Address{}, gas, ErrNonceUintOverflow
    }
    evm.StateDB.SetNonce(caller, nonce+1, tracing.NonceChangeContractCreator)
```
注意是设置caller 的nonce，因为这也算是一次交易。
nonce 是world state 中的一个字段，每次交易+1，用来防止重放攻击，伪造交易签名。


### 添加access list

```go
    if evm.chainRules.IsEIP2929 {
        evm.StateDB.AddAddressToAccessList(address)
    }
```
见： [[7. Contract Creation#账户状态更新]]
ACCESS list概念见 [[EIP-2929]]

注意，AccessList 的调用在 statedb.snapshot [[journal.go#snapshot]] 之前，即创建失败也会添加。

### 异常条件-目标账户地址非空
```go
	// Ensure there's no existing contract already at the designated address.
	// Account is regarded as existent if any of these three conditions is met:
	// - the nonce is non-zero
	// - the code is non-empty
	// - the storage is non-empty
	contractHash := evm.StateDB.GetCodeHash(address)
	storageRoot := evm.StateDB.GetStorageRoot(address)
	if evm.StateDB.GetNonce(address) != 0 ||
		(contractHash != (common.Hash{}) && contractHash != types.EmptyCodeHash) || // non-empty code
		(storageRoot != (common.Hash{}) && storageRoot != types.EmptyRootHash) { // non-empty storage
		return nil, common.Address{}, 0, ErrContractAddressCollision
	}
```

### 创建和初始化一个合约账户
```go
    evm.StateDB.CreateContract(address)
    if evm.chainRules.IsEIP158 {
        evm.StateDB.SetNonce(address, 1, tracing.NonceChangeNewContract)
    }
    ...
    evm.Context.Transfer(evm.StateDB, caller, address, value)
```

### 调用 initNewContract
#### 入参说明
```go
contract := NewContract(caller, address, value, gas, evm.jumpDests)
contract.SetCallCode(common.Hash{}, code)
contract.IsDeployment = true


```
contract 中，包括了调用的账号，创建合约的目标账号，balance 是来的转账，gas 是交易中的gas limit 扣除后剩余的。
code 来自上级传入，注意这里code hash 是一个空值，这只是一个约定，表示此时contract 还在创建中。详见：[[valid jump dest]]。当然了，此处调用set code 的目的肯定是给合约的初始化代码作为input。

#### init code 执行
```go
ret, err := evm.interpreter.Run(contract, nil, false)
```

#### 检查和gas
```go
	// Check whether the max code size has been exceeded, assign err if the case.
	if evm.chainRules.IsEIP158 && len(ret) > params.MaxCodeSize {
		return ret, ErrMaxCodeSizeExceeded
	}

	// Reject code starting with 0xEF if EIP-3541 is enabled.
	if len(ret) >= 1 && ret[0] == 0xEF && evm.chainRules.IsLondon {
		return ret, ErrInvalidCode
	}
	createDataGas := uint64(len(ret)) * params.CreateDataGas      
	contract.UseGas(createDataGas, ...)

```

类似于：
$$
\lor \|o\| > 24576 
\lor o[0] = \text{0xef}
$$
以及：
$$
g^{**} < c
$$

#### 把输出写入db
```go
// ret, err := evm.interpreter.Run(contract, nil, false)
evm.StateDB.SetCode(address, ret)
```
即下面说的第三种情况：

$$
\sigma' = 
\begin{cases}
\sigma & \text{if } F \lor \sigma^{**} = \emptyset \\
\sigma^{**} \text{ except } \sigma'[a] = \emptyset & \text{if } \text{DEAD}(\sigma^{**}, a) \\
\sigma^{**} \text{ except } \sigma'[a]_c = \text{KEC}(o) & \text{otherwise}
\end{cases}
$$
对比call，call 的输出o 仅仅是写入evm.interpreter，等待 `RETURNDATACOPY` 的 Operator 来使用它。

# opCreate
## 流程
### 通用检查
```go
	if interpreter.readOnly {
		return nil, ErrWriteProtection
	}
```

### gas 相关
```go
if interpreter.evm.chainRules.IsEIP150 {
    gas -= gas / 64
}

scope.Contract.UseGas(scope.Contract.Gas, interpreter.evm.Config.Tracer, tracing.GasChangeCallContractCreation)
...
// 执行 create
scope.Contract.RefundGas(returnGas, interpreter.evm.Config.Tracer, tracing.GasChangeCallLeftOverRefunded)
```
输入的gas ，先扣除1/64，再在创建过程中按需扣除。
其中：
按照 **EIP-150** 规则，最多只能使用当前 gas 的 `63/64`，防止恶意调用者耗尽所有 gas。确保调用者有足够 gas 能处理合约返回，避免无法清理栈、内存等。

### 栈相关
```go
value        = scope.Stack.pop()
offset, size = scope.Stack.pop(), scope.Stack.pop()
//Note: input        = scope.Memory.GetCopy(offset.Uint64(), size.Uint64()) 

...
// 执行 create

if interpreter.evm.chainRules.IsHomestead && suberr == ErrCodeStoreOutOfGas {
    stackvalue.Clear()
} else if suberr != nil && suberr != ErrCodeStoreOutOfGas {
    stackvalue.Clear()
} else {
    stackvalue.SetBytes(addr.Bytes())
}
scope.Stack.push(&stackvalue)
```

- 如果合约创建失败，压栈 0。
- 如果成功，压栈合约地址。
注：上面的代码有点复杂，但意思就是：如果是 `ErrCodeStoreOutOfGas` 错误，只能在 Homestead 后视为失败

### 回传数据处理
```go
// 执行 create
if suberr == ErrExecutionReverted {
	interpreter.returnData = res
	return res, nil
}
interpreter.returnData = nil
return nil, nil
```
- 如果是 REVERT，设置 returnData，可以被父合约通过 `RETURNDATACOPY` 获取。
- 否则清空 returnData。

