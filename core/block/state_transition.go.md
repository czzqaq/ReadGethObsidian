# ApplyMessage

直接调用了stateTransition.execute 函数。
为了理解这部分代码，需要先掌握[[6. Transaction Execution]]
构造好环境后，直接进入 execute()

## precheck
```go
func (st *stateTransition) execute() (*ExecutionResult, error) {
	// First check this message satisfies all consensus rules before
	// applying the message. The rules include these clauses
	//
	// 1. the nonce of the message caller is correct
	// 2. caller has enough balance to cover transaction fee(gaslimit * gasprice)
	// 3. the amount of gas required is available in the block
	// 4. the purchased gas is enough to cover intrinsic usage
	// 5. there is no overflow when calculating intrinsic gas
	// 6. caller has enough balance to cover asset transfer for **topmost** call

	// Check clauses 1-3, buy gas if everything is correct
	if err := st.preCheck(); err != nil {
		return nil, err
	}

```

即对应了：

| 检查项编号 | 检查内容                                | 来源 / EIP | preCheck 是否处理                    |
| ----- | ----------------------------------- | -------- | -------------------------------- |
| 1     | 有合法 sender                          | 基础       | 是                                |
| 2     | sender 非合约账户（EOA）                   | EIP-3607 | 是                                |
| 3     | nonce 与状态匹配                         | 基础       | 是                                |
| 4     | gas limit ≥ intrinsic gas           | 基础       | 否，见[[#intrinsic gas 检查]]         |
| 5     | balance ≥ up-front cost             | 基础       | 否，见[[#余额充足]]                     |
| 6     | gasFeeCap ≥ baseFee                 | EIP-1559 | 是                                |
| 7     | init 长度 < 49152                     | EIP-3860 | 否，见 [[#init code size 限制]]       |
| 8     | gasLimit + usedGas ≤ blockGasLimit  | 区块层检查    | 否                                |
| 9     | maxFeePerGas ≥ maxPriorityFeePerGas | EIP-1559 | 是                                |
| 10    | blob tx 必须指定 To                     | EIP-4844 | 是，额外                             |
| 11    | blob hashes 非空且合法                   | EIP-4844 | 是，额外                             |
| 12    | blobGasFeeCap ≥ blobBaseFee         | EIP-4844 | 是，额外                             |
| 13    | EIP-7702 授权列表合法性                    | EIP-7702 | 是，额外                             |
| 14    | Floor Data Gas 检查                   | EIP-7623 | 否，见[[#Floor Data Gas(EIP-7623)]] |
## intrinsicGas

>- 交易数据字段的消耗，如果是全0可以便宜一点，T_i 用于contract creation，T_d 用于智能合约调用。
>- 合约创建时，有一笔基础费用, 且根据 init 数据段的长度额外收费, 收费公式 R 详见 [[#初始化代码成本函数 $R$]]
>- 交易基本开销
>- 根据 EIP-2930，交易的 AccessList 中，全部地址上储存键值的总数量，每个产生一笔，以及每个账户的储存。


### 基础交易
```go
if isContractCreation && isHomestead {
    gas = params.TxGasContractCreation
} else {
    gas = params.TxGas
}
```

其中，isHomestead 是一个很久之前的分支，在此之后，都区别计算创建和message call 的transaction。


### input data 计算gas
```go
z := uint64(bytes.Count(data, []byte{0}))
nz := dataLen - z
nonZeroGas := params.TxDataNonZeroGasFrontier
if isEIP2028 {
    nonZeroGas = params.TxDataNonZeroGasEIP2028
}
gas += nz * nonZeroGas
gas += z * params.TxDataZeroGas
```
isEIP2028 规定了0字段用更少的gas。
对应：
$$\sum_{i \in T_i, T_d}
\begin{cases}
G_{\text{txdatazero}} & \text{if } i = 0 \\
G_{\text{txdatanonzero}} & \text{otherwise}
\end{cases}$$

### init code 附加gas（EIP-3860）
```go
if isContractCreation && isEIP3860 {
    lenWords := toWordSize(dataLen)
    gas += lenWords * params.InitCodeWordGas
}
```
对应：
$$+ R(\lVert T_i \rVert) \quad \text{if } T_t = \emptyset$$

### access list 相关（EIP-2930）
```go
if accessList != nil {
    gas += uint64(len(accessList)) * params.TxAccessListAddressGas
    gas += uint64(accessList.StorageKeys()) * params.TxAccessListStorageKeyGas
}
```
对应：
$$+ \sum_{j=0}^{\lVert T_A \rVert - 1} \left( G_{\text{accesslistaddress}} + \lVert T_A[j]_s \rVert \cdot G_{\text{accessliststorage}} \right)$$

### 授权账户 gas (EIP-7702)
```go
if authList != nil {
    gas += uint64(len(authList)) * params.CallNewAccountGas
}
```
- 每个授权地址收取 `CallNewAccountGas`


## 下一阶段验证
### intrinsic gas 检查
```go
gas, err := IntrinsicGas(...)
if st.gasRemaining < gas {
	return nil, fmt.Errorf("%w: have %d, want %d", ErrIntrinsicGas, st.gasRemaining, gas)
}
```

### Floor Data Gas(EIP-7623)
```go
if rules.IsPrague {
	floorDataGas, err = FloorDataGas(msg.Data)
	if msg.GasLimit < floorDataGas {
		return nil, fmt.Errorf("%w: have %d, want %d", ErrFloorDataGas, msg.GasLimit, floorDataGas)
	}
}
```

### 余额充足
```go
value, overflow := uint256.FromBig(msg.Value)
if overflow {
	return nil, fmt.Errorf("%w: address %v", ErrInsufficientFundsForTransfer, msg.From.Hex())
}
if !value.IsZero() && !st.evm.Context.CanTransfer(st.state, msg.From, value) {
	return nil, fmt.Errorf("%w: address %v", ErrInsufficientFundsForTransfer, msg.From.Hex())
}
```

### init code size 限制
```go
if rules.IsShanghai && contractCreation && len(msg.Data) > params.MaxInitCodeSize {
	return nil, fmt.Errorf("%w: code size %v limit %v", ErrMaxInitCodeSizeExceeded, len(msg.Data), params.MaxInitCodeSize)
}
```

## prepare
### 访问列表更新(EIP-4762)
```go
if rules.IsEIP4762 {
	st.evm.AccessEvents.AddTxOrigin(msg.From)
	if targetAddr := msg.To; targetAddr != nil {
		st.evm.AccessEvents.AddTxDestination(*targetAddr, msg.Value.Sign() != 0)
	}
}
```
EIP-4762 引入的 `AccessEvents`，影响状态访问追踪，非强制性验证
这段代码插在了检查代码中。


### Prepare
这段代码对应了 [[6. Transaction Execution#构造子状态 $A *$]]
#### 得到 A_0

```go
al := newAccessList()
```
初始化 / 清空访问列表（Access List），即定义了一个空的 子状态
$$
A_0 \equiv (\emptyset, (), \emptyset, 0, \pi, \emptyset)
$$
#### A_K 计算

$A_K$：**被访问的存储键集合**

```go
for _, el := range list {
	al.AddAddress(el.Address) // 这行是对 A_a 的计算
	for _, key := range el.StorageKeys {
		al.AddSlot(el.Address, key)
	}
}
```
对应了
$$A^*_K \equiv \bigcup_{E \in T_A} \{ (E_a, E_s[i]) \mid \forall i < \lVert E_s \rVert \}$$

#### A_a AccessList 中 account 的更新
子状态中的访问存储键集合，来自于 access list 中每个地址对应的 storage keys。


$$
A^*_a \equiv
\begin{cases}
a \cup \{T_t\} & \text{if } T_t \ne \emptyset \\
a & \text{otherwise}
\end{cases}$$

$$
a \equiv A^0_a \cup \{S(T)\} \cup H_c \cup \bigcup_{E \in T_A} \{E_a\}$$

```go
al.AddAddress(sender) // S(T)
if dst != nil {  // dst 是T_t，to 地址
	al.AddAddress(*dst)
}
if rules.IsShanghai { // H_c
	al.AddAddress(coinbase)
}
for _, addr := range precompiles { // precompiles 是 A_a 中初始化的 pi
	al.AddAddress(addr)
}

for _, el := range list {
    al.AddAddress(el.Address)
//    for _, key := range el.StorageKeys {
//        al.AddSlot(el.Address, key)
//    }
}
```

#### 总结
| 项目               | 公式符号                | Prepare() 中的代码                                    | 说明                     |
| ---------------- | ------------------- | ------------------------------------------------- | ---------------------- |
| 初始 warm 地址集合     | \( A^0_a \)         |  precompiles                                      | precompiles 定义于 EVM 协议 |
| sender 地址        | \( S(T) \)          | `al.AddAddress(sender)`                           | 明确添加                   |
| To 地址（如果存在）      | \( T_t \)           | `if dst != nil { al.AddAddress(*dst) }`           | 非 CREATE 交易才有          |
| coinbase         | \( H_c \)           | `if rules.IsShanghai { al.AddAddress(coinbase) }` | EIP-3651 引入            |
| access list 地址   | \( E_a \in T_A \)   | `al.AddAddress(el.Address)`                       | 来自 EIP-2930            |
| access list 的存储键 | \( (E_a, E_s[i]) \) | `al.AddSlot(el.Address, key)`                     | 来自 EIP-2930            |

### transientStorage 修改
#### 清空

```go
func (s *StateDB) Prepare(rules params.Rules, sender, coinbase common.Address, dst *common.Address, precompiles []common.Address, list types.AccessList) {
    if rules.IsEIP2929 {
    ...
    }
    s.transientStorage = newTransientStorage()
}
```
这里对 transientStorage 做了清空。在黄皮书中没有这步，它反而说：
>我们首先定义**检查点状态（checkpoint state）** $\sigma_0$，它是在交易执行前对原状态 $\sigma$ 作出两点修改后的结果: 扣除交易发送者的余额 $T_g \cdot p$（用于支付 gas），并将其 nonce 加 1。

#### 设置nonce
```go
st.state.SetNonce(msg.From, st.state.GetNonce(msg.From)+1, tracing.NonceChangeEoACall)
```
$$\sigma_0[S(T)]_n = \sigma[S(T)]_n + 1$$

#### gas 处理
