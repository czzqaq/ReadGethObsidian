这部分直接让chatgpt 帮忙分析的。
gas 消耗的另一部分见：
[[state_transition.go#intrinsicGas]]


## 🧠 Ethereum EVM 运行时 Gas 花费笔记

---

### 📌 1. **内存扩展 Memory Gas Cost**

```go
memoryGasCost(mem *Memory, newMemSize uint64) (uint64, error)
```

#### ✅ 核心逻辑：

- 内存以 **32 字节为单位（word）** 扩展。
    
- Gas 花费公式：
    
    ```
    gas = linear_cost + quadratic_cost
    linear_cost = words * params.MemoryGas
    quadratic_cost = (words^2) / params.QuadCoeffDiv
    ```
    
- 只为新增内存区域计费（差值）。
    
- 记录上次 gas 消耗 `mem.lastGasCost`，避免重复计费。
    

#### ✅ 特点：

- 内存扩展越大，成本越高（非线性）。
- 有最大限制：防止溢出 `newMemSize > 0x1FFFFFFFE0`。

---

### 📌 2. **数据复制类指令 Gas**

```go
memoryCopierGas(stackpos int)
```

#### 涉及指令：

- `CALLDATACOPY`, `CODECOPY`, `EXTCODECOPY`, `RETURNDATACOPY`, `MCOPY`

#### Gas 组成：

1. **内存扩展成本**（调用 `memoryGasCost`）
2. **复制成本**：按 word 计算的复制成本

```go
gas += toWordSize(bytesToCopy) * params.CopyGas
```

---

### 📌 3. **SSTORE 的 Gas 计算**

#### 分为两个版本：

- `gasSStore`: 旧规则（Petersburg 或非 Constantinople）
- `gasSStoreEIP2200`: EIP-2200 新规则（Istanbul 及以后）

---

#### 📍 EIP-2200 下的逻辑复杂度高，处理了：

- 当前值是否等于新值 → No-op
- 当前值是否等于 original（提交状态）→ 是否 dirty slot
- 是否从零变为非零、或反之 → 判断是否需要 refund
- refund 的加减逻辑非常细致，防止过度退款

#### ✅ 关键 refund 参数：

```go
params.SstoreClearsScheduleRefundEIP2200
params.SstoreSetGasEIP2200
params.SstoreResetGasEIP2200
params.SloadGasEIP2200
```

> 💡 **refund 机制非常重要，会直接影响最终交易成本**

---

### 📌 4. **CREATE / CREATE2 / EIP-3860 Gas**

#### 涉及函数：

- `gasCreate`
- `gasCreate2`
- `gasCreateEip3860`
- `gasCreate2Eip3860`

#### Gas 组成：

1. 内存扩展 gas
2. `CREATE2` 还需要考虑 `keccak256(init_code)`
3. `EIP-3860` 规定了 `init_code` 长度限制和字长 gas：

```go
InitCodeWordGas * ((size + 31) / 32)
```

> 防止 DoS 攻击：太长的 init code 会导致巨大开销

---

### 📌 5. **日志指令 LOGX 的 Gas**

```go
makeGasLog(n uint64)
```

#### Gas 组成：

1. 内存扩展
2. 固定日志成本 `params.LogGas`
3. 每个 topic 的成本 `params.LogTopicGas`
4. 日志数据长度 × `params.LogDataGas`

---

### 📌 6. **Keccak256 的 Gas**

```go
gasKeccak256
```

#### Gas 组成：

1. 内存扩展
2. 哈希成本 = 每 word 成本 × word 数量

```go
wordGas = toWordSize(length) * params.Keccak256WordGas
```

---

### 📌 7. **CALL / CALLCODE / DELEGATECALL / STATICCALL**

#### 统一结构：

- 内存扩展
- 存在 value 转账时加上固定成本
- 如果目标地址是新账户，有 `CallNewAccountGas`
- 最终要调用 `callGas(...)` 来决定实际可用 gas（考虑 EIP-150）

#### 特别注意：

- `EIP-4762` 引入了 `AccessEvents.ValueTransferGas(...)` 动态计算的转账成本
- `callGas(...)` 的计算非常关键，影响子调用能用多少 gas

---

### 📌 8. **SELFDESTRUCT 的 Gas**

#### 逻辑：

- 固定成本 `params.SelfdestructGasEIP150`
- 若目标地址是空账户，则加上 `CreateBySelfdestructGas`
- 若合约第一次 selfdestruct，会触发 `AddRefund(params.SelfdestructRefundGas)`

---

### 📌 9. **EXP（指数运算）**

#### 两种版本：

- `gasExpFrontier`: `params.ExpByteFrontier`
- `gasExpEIP158`: `params.ExpByteEIP158`

#### Gas 组成：

```go
params.ExpGas + expByteLen * params.ExpByteGas
```

其中 `expByteLen` 是指数的字节长度。

---

## 🧾 总结：Gas 评估的核心原则

|类型|影响因素|Refund 支持|非线性|特殊规则|
|---|---|---|---|---|
|内存扩展|新增的内存长度（按 word）|否|✅|是|
|数据复制|复制长度 + 内存扩展|否|❌|否|
|SSTORE|当前值、原始值、目标值关系|✅|❌|✅（EIP-2200）|
|CREATE/2|init code 长度、hash|否|❌|✅（EIP-3860）|
|LOG|topic 数量、data 长度|否|❌|否|
|CALL 类|value 转账、目标是否新账户等|否|❌|✅（EIP-150）|
|SELFDESTRUCT|余额是否转出、是否首次调用|✅|❌|✅|
|EXP|指数字节长度|否|✅|是|
