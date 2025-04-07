
# 指令汇总

下面这份表格，汇总了当前 `go-ethereum` 代码中（基于所附代码）的 **JumpTable** 主要关键信息，包括：

- **指令名**  
- **指令值**（即在 `JumpTable[256]` 中对应的索引，十六进制）  
- **Execute 函数**（对应 `opXXX` 或 `makeXXX` 等 Go 函数）  
- **动态 Gas 函数**（若无则填 `NA`）  
- **pop 数**（从栈顶弹出的操作数个数）  
- **栈变化数**（即执行完指令后，栈净增加量，可以为负数。例如 `pop=2, push=1` 则栈变化 = `1 - 2 = -1`）  
- **从此 Fork 起支持**（最早从哪个分叉开始支持，如果该指令自创世就有，则为 `Frontier`；若在后续 Fork 中新增或更名，则从该分叉开始）  

## Frontier 阶段指令

以下指令在 Frontier 便已存在或定义。

### 基本计算

| 指令名        | 指令值 (Hex) | Execute 函数     | 动态Gas函数          | pop 数 | 栈变化 | 从此 Fork 起支持 |
| :--------- | :-------: | :------------- | :--------------- | :---: | :-: | :---------: |
| STOP       |   0x00    | `opStop`       | NA               |   0   |  0  |  Frontier   |
| ADD        |   0x01    | `opAdd`        | NA               |   2   | -1  |  Frontier   |
| MUL        |   0x02    | `opMul`        | NA               |   2   | -1  |  Frontier   |
| SUB        |   0x03    | `opSub`        | NA               |   2   | -1  |  Frontier   |
| DIV        |   0x04    | `opDiv`        | NA               |   2   | -1  |  Frontier   |
| SDIV       |   0x05    | `opSdiv`       | NA               |   2   | -1  |  Frontier   |
| MOD        |   0x06    | `opMod`        | NA               |   2   | -1  |  Frontier   |
| SMOD       |   0x07    | `opSmod`       | NA               |   2   | -1  |  Frontier   |
| ADDMOD     |   0x08    | `opAddmod`     | NA               |   3   | -2  |  Frontier   |
| MULMOD     |   0x09    | `opMulmod`     | NA               |   3   | -2  |  Frontier   |
| EXP        |   0x0A    | `opExp`        | `gasExpFrontier` |   2   | -1  |  Frontier   |
| SIGNEXTEND |   0x0B    | `opSignExtend` | NA               |   2   | -1  |  Frontier   |
|            |           |                |                  |       |     |             |

| 指令名       | 指令值 (Hex) | Execute 函数      | 动态Gas函数            | pop 数 | 栈变化  | 从此 Fork 起支持 |
|:-------------|:-----------:|:------------------|:-----------------------|:------:|:-------:|:---------------:|
| LT           | 0x10        | `opLt`            | NA                    | 2      | -1      | Frontier        |
| GT           | 0x11        | `opGt`            | NA                    | 2      | -1      | Frontier        |
| SLT          | 0x12        | `opSlt`           | NA                    | 2      | -1      | Frontier        |
| SGT          | 0x13        | `opSgt`           | NA                    | 2      | -1      | Frontier        |
| EQ           | 0x14        | `opEq`            | NA                    | 2      | -1      | Frontier        |
| ISZERO       | 0x15        | `opIszero`        | NA                    | 1      | 0       | Frontier        |
| AND          | 0x16        | `opAnd`           | NA                    | 2      | -1      | Frontier        |
| OR           | 0x17        | `opOr`            | NA                    | 2      | -1      | Frontier        |
| XOR          | 0x18        | `opXor`           | NA                    | 2      | -1      | Frontier        |
| NOT          | 0x19        | `opNot`           | NA                    | 1      | 0       | Frontier        |
| BYTE         | 0x1A        | `opByte`          | NA                    | 2      | -1      | Frontier        |
### KECCAK256

| 指令名       | 指令值 (Hex) | Execute 函数    | 动态Gas函数        | pop 数 | 栈变化 | 从此 Fork 起支持 |
| :-------- | :-------: | :------------ | :------------- | :---: | :-: | :---------: |
| KECCAK256 |   0x20    | `opKeccak256` | `gasKeccak256` |   2   | -1  |  Frontier   |
### 世界状态相关

| 指令名          | 指令值 (Hex) | Execute 函数       | 动态Gas函数               | pop 数 | 栈变化 | 从此 Fork 起支持 |
| :----------- | :-------: | :--------------- | :-------------------- | :---: | :-: | :---------: |
| ADDRESS      |   0x30    | `opAddress`      | NA                    |   0   | +1  |  Frontier   |
| BALANCE      |   0x31    | `opBalance`      | NA (前期 `constantGas`) |   1   |  0  |  Frontier   |
| ORIGIN       |   0x32    | `opOrigin`       | NA                    |   0   | +1  |  Frontier   |
| CALLER       |   0x33    | `opCaller`       | NA                    |   0   | +1  |  Frontier   |
| CALLVALUE    |   0x34    | `opCallValue`    | NA                    |   0   | +1  |  Frontier   |
| CALLDATALOAD |   0x35    | `opCallDataLoad` | NA                    |   1   |  0  |  Frontier   |
| CALLDATASIZE |   0x36    | `opCallDataSize` | NA                    |   0   | +1  |  Frontier   |
| CALLDATACOPY |   0x37    | `opCallDataCopy` | `gasCallDataCopy`     |   3   | -3  |  Frontier   |
| CODESIZE     |   0x38    | `opCodeSize`     | NA                    |   0   | +1  |  Frontier   |
| CODECOPY     |   0x39    | `opCodeCopy`     | `gasCodeCopy`         |   3   | -3  |  Frontier   |
| GASPRICE     |   0x3A    | `opGasprice`     | NA                    |   0   | +1  |  Frontier   |
| EXTCODESIZE  |   0x3B    | `opExtCodeSize`  | `gasExtCodeCopy` (变体) |   1   |  0  |  Frontier   |
| EXTCODECOPY  |   0x3C    | `opExtCodeCopy`  | `gasExtCodeCopy`      |   4   | -4  |  Frontier   |

| 指令名           | 指令值 (Hex) | Execute 函数     | 动态Gas函数 | pop 数 | 栈变化 | 从此 Fork 起支持 |
| :------------ | :-------: | :------------- | :------ | :---: | :-: | :---------: |
| BLOCKHASH     |   0x40    | `opBlockhash`  | NA      |   1   |  0  |  Frontier   |
| COINBASE      |   0x41    | `opCoinbase`   | NA      |   0   | +1  |  Frontier   |
| TIMESTAMP     |   0x42    | `opTimestamp`  | NA      |   0   | +1  |  Frontier   |
| NUMBER        |   0x43    | `opNumber`     | NA      |   0   | +1  |  Frontier   |
| DIFFICULTY(旧) |   0x44    | `opDifficulty` | NA      |   0   | +1  |  Frontier   |
| GASLIMIT      |   0x45    | `opGasLimit`   | NA      |   0   | +1  |  Frontier   |
### 逻辑控制

| 指令名      | 指令值 (Hex) | Execute 函数   | 动态Gas函数      | pop 数 | 栈变化 | 从此 Fork 起支持 |
| :------- | :-------: | :----------- | :----------- | :---: | :-: | :---------: |
| POP      |   0x50    | `opPop`      | NA           |   1   | -1  |  Frontier   |
| MLOAD    |   0x51    | `opMload`    | `gasMLoad`   |   1   |  0  |  Frontier   |
| MSTORE   |   0x52    | `opMstore`   | `gasMStore`  |   2   | -2  |  Frontier   |
| MSTORE8  |   0x53    | `opMstore8`  | `gasMStore8` |   2   | -2  |  Frontier   |
| SLOAD    |   0x54    | `opSload`    | NA           |   1   |  0  |  Frontier   |
| SSTORE   |   0x55    | `opSstore`   | `gasSStore`  |   2   | -2  |  Frontier   |
| JUMP     |   0x56    | `opJump`     | NA           |   1   | -1  |  Frontier   |
| JUMPI    |   0x57    | `opJumpi`    | NA           |   2   | -2  |  Frontier   |
| PC       |   0x58    | `opPc`       | NA           |   0   | +1  |  Frontier   |
| MSIZE    |   0x59    | `opMsize`    | NA           |   0   | +1  |  Frontier   |
| GAS      |   0x5A    | `opGas`      | NA           |   0   | +1  |  Frontier   |
| JUMPDEST |   0x5B    | `opJumpdest` | NA           |   0   |  0  |  Frontier   |
|          |           |              |              |       |     |             |

### PUSH 系列 (0x60 ~ 0x7F)

所有 PUSH 系列指令均为**Frontier**时就已定义，`execute` 函数在 go-ethereum 中一般用 `makePush(size, size)` 生成，Gas 皆为 `GasFastestStep`，无动态 Gas。

| 指令名  | 指令值 (Hex) | Execute 函数        | pop 数 | 栈变化 | 从此 Fork 起支持 |
|:--------|:-----------:|:--------------------|:------:|:------:|:---------------:|
| PUSH1   | 0x60        | `opPush1` 或 `makePush(1,1)`   | 0   | +1  | Frontier        |
| PUSH2   | 0x61        | `makePush(2,2)`     | 0      | +1     | Frontier        |
| PUSH3   | 0x62        | `makePush(3,3)`     | 0      | +1     | Frontier        |
| ...     | ...         | ...                | ...    | ...    | Frontier        |
| PUSH32  | 0x7F        | `makePush(32,32)`   | 0      | +1     | Frontier        |

### DUP 系列 (0x80 ~ 0x8F)

| 指令名  | 指令值 (Hex) | Execute 函数    | pop 数 (最小栈深) | 栈变化 | 从此 Fork 起支持 |
|:--------|:-----------:|:----------------|:------------------:|:------:|:---------------:|
| DUP1    | 0x80        | `makeDup(1)`    | 1                  | +1     | Frontier        |
| DUP2    | 0x81        | `makeDup(2)`    | 2                  | +1     | Frontier        |
| DUP3    | 0x82        | `makeDup(3)`    | 3                  | +1     | Frontier        |
| ...     | ...         | ...            | ...                | ...    | Frontier        |
| DUP16   | 0x8F        | `makeDup(16)`   | 16                 | +1     | Frontier        |

### SWAP 系列 (0x90 ~ 0x9F)

| 指令名  | 指令值 (Hex) | Execute 函数  | pop 数 (最小栈深) | 栈变化 | 从此 Fork 起支持 |
|:--------|:-----------:|:--------------|:------------------:|:------:|:---------------:|
| SWAP1   | 0x90        | `opSwap1`     | 2                  | 0      | Frontier        |
| SWAP2   | 0x91        | `opSwap2`     | 3                  | 0      | Frontier        |
| ...     | ...         | ...          | ...                | ...    | Frontier        |
| SWAP16  | 0x9F        | `opSwap16`    | 17                 | 0      | Frontier        |

### LOG 系列 (0xA0 ~ 0xA4)

| 指令名  | 指令值 (Hex) | Execute 函数    | 动态Gas函数         | pop 数 | 栈变化 | 从此 Fork 起支持 |
|:--------|:-----------:|:----------------|:--------------------|:------:|:------:|:---------------:|
| LOG0    | 0xA0        | `makeLog(0)`    | `makeGasLog(0)`     | 2      | -2     | Frontier        |
| LOG1    | 0xA1        | `makeLog(1)`    | `makeGasLog(1)`     | 3      | -3     | Frontier        |
| LOG2    | 0xA2        | `makeLog(2)`    | `makeGasLog(2)`     | 4      | -4     | Frontier        |
| LOG3    | 0xA3        | `makeLog(3)`    | `makeGasLog(3)`     | 5      | -5     | Frontier        |
| LOG4    | 0xA4        | `makeLog(4)`    | `makeGasLog(4)`     | 6      | -6     | Frontier        |

### CREATE / CALL / RETURN / SELFDESTRUCT 等 (0xF0 ~ 0xFF)

| 指令名        | 指令值 (Hex) | Execute 函数      | 动态Gas函数       | pop 数 | 栈变化 | 从此 Fork 起支持 |
|:--------------|:-----------:|:------------------|:------------------|:------:|:------:|:---------------:|
| CREATE        | 0xF0        | `opCreate`        | `gasCreate`       | 3      | -2     | Frontier        |
| CALL          | 0xF1        | `opCall`          | `gasCall`         | 7      | -6     | Frontier        |
| CALLCODE      | 0xF2        | `opCallCode`      | `gasCallCode`     | 7      | -6     | Frontier        |
| RETURN        | 0xF3        | `opReturn`        | `gasReturn`       | 2      | -2     | Frontier        |
| DELEGATECALL  | 0xF4        | *Homestead后添加* | `gasDelegateCall` | 6      | -5     | **Homestead**    |
| CREATE2       | 0xF5        | `opCreate2`       | `gasCreate2`      | 4      | -3     | **Constantinople** |
| …            |   …         |    …               |  …                |  …     |  …     |   …             |
| STATICCALL    | 0xFA        | `opStaticCall`    | `gasStaticCall`   | 6      | -5     | **Byzantium**    |
| REVERT        | 0xFD        | `opRevert`        | `gasRevert`       | 2      | -2     | **Byzantium**    |
| INVALID       | 0xFE        | `opUndefined`     | NA                | 0      | 0      | Frontier（一直未定义） |
| SELFDESTRUCT  | 0xFF        | `opSelfdestruct`  | `gasSelfdestruct` | 1      | -1     | Frontier        |

---

## 主要在后续分叉新增或改动的指令

以下为在 **Frontier** 之后引入或显著修改的指令（或对应常量）。一些旧指令（如 `BALANCE`, `EXTCODESIZE`, `SLOAD` 等）会在新分叉中只调整 Gas 不同，这里仅列新分叉“新增”的操作码或功能要点。

1. **Homestead**  
   - `DELEGATECALL (0xF4)`  
     - Execute 函数：`opDelegateCall`  
     - 动态 Gas 函数：`gasDelegateCall`  
     - pop 数 6，栈变化 -5  
     - 从 Homestead 起支持

2. **Tangerine Whistle (EIP-150)** / **Spurious Dragon (EIP-158)**  
   - 主要是修改了若干已有指令的 Gas 公式，例如 `BALANCE`, `EXTCODESIZE`, `SLOAD`, `EXP` 等；并未新增新的操作码。

3. **Byzantium**  
   - `STATICCALL (0xFA)`  
     - Execute 函数：`opStaticCall`  
     - 动态 Gas 函数：`gasStaticCall`  
     - pop 数 6，栈变化 -5  
     - 从 Byzantium 起支持
   - `RETURNDATASIZE (0x3D)` / `RETURNDATACOPY (0x3E)` 在实际以太坊里是 Byzantium 引入。（在 `go-ethereum` 代码中是 0x3d / 0x3e）  
   - `REVERT (0xFD)`

4. **Constantinople**  
   - `CREATE2 (0xF5)`  
   - `SHL (0x1B)`, `SHR (0x1C)`, `SAR (0x1D)`  
   - `EXTCODEHASH (0x3F)`

5. **Istanbul**  
   - `CHAINID (0x46)`, `SELFBALANCE (0x47)` 等（此处在源码里由 `enable1344` / `enable1884` / `enable2200` 引入）  
   - 代码中也可看到对 `BALANCE` / `SLOAD` 的 Gas 再次修改。

6. **Berlin**  
   - 主要是 EIP-2929 调整了状态访问的 Gas 成本，无新增操作码。  

7. **London**  
   - `BASEFEE (0x48)` (opcode)  
     - 从 London 起支持

8. **Merge**  
   - 将 **DIFFICULTY (0x44)** 替换成 `PREVRANDAO (0x44)`  
     - 从 Merge 起，0x44 opcode 的执行变为 `opRandom`。

9. **Shanghai**  
   - `PUSH0 (0x5F)`  
     - 从 Shanghai 起支持

10. **Cancun** / **Prague** / **EOF**  
   - 代码中还有一些如 `BLOBHASH (0x49?)`、`MCOPY`、`BLOBBASEFEE`、`PUSH0` 之后的一些提案 opcode，属于尚未正式主网激活或属于后续硬分叉（如 Cancun、Prague、EOF）的指令集演进，具体见对应 EIP 提案和 `go-ethereum` 源码。

---

## 示范性汇总（含后续 EIP 新增）

以下仅列出部分在后续分叉中新增的关键 opcode 完整行示例：

| 指令名       | 指令值 (Hex) | Execute 函数       | 动态Gas函数        | pop 数 | 栈变化 | 从此 Fork 起支持    |
|:------------|:-----------:|:-------------------|:-------------------|:------:|:------:|:-------------------:|
| DELEGATECALL| 0xF4        | `opDelegateCall`   | `gasDelegateCall`  | 6      | -5     | Homestead           |
| STATICCALL  | 0xFA        | `opStaticCall`     | `gasStaticCall`    | 6      | -5     | Byzantium           |
| REVERT      | 0xFD        | `opRevert`         | `gasRevert`        | 2      | -2     | Byzantium           |
| CREATE2     | 0xF5        | `opCreate2`        | `gasCreate2`       | 4      | -3     | Constantinople      |
| SHL         | 0x1B        | `opSHL`            | NA                 | 2      | -1     | Constantinople      |
| SHR         | 0x1C        | `opSHR`            | NA                 | 2      | -1     | Constantinople      |
| SAR         | 0x1D        | `opSAR`            | NA                 | 2      | -1     | Constantinople      |
| EXTCODEHASH | 0x3F        | `opExtCodeHash`    | NA / 常量Gas       | 1      | 0      | Constantinople      |
| CHAINID     | 0x46        | `opChainId`        | NA (常量Gas)       | 0      | +1     | Istanbul            |
| BASEFEE     | 0x48        | `opBaseFee`        | NA (常量Gas)       | 0      | +1     | London              |
| PREVRANDAO  | 0x44        | `opRandom`         | NA (常量Gas)       | 0      | +1     | Merge (替换Difficulty) |
| PUSH0       | 0x5F        | `opPush0`          | NA                 | 0      | +1     | Shanghai            |

# 代码说明
## opcode 的映射
```go
type JumpTable [256]*operation`
```

每个字节码（0x00 - 0xFF）都映射到一个 `*operation`，即 EVM 指令的定义。

## fork 演变链和`newXxxInstructionSet()`
Frontier → Homestead → TangerineWhistle → SpuriousDragon → Byzantium → Constantinople → Istanbul → Berlin → London → Merge → Shanghai → Cancun → Prague → Verkle

每一个 `newXxxInstructionSet()` 函数：

- 复制前一阶段的 `JumpTable`
- 对新加入的 EIP 特性进行启用（如 `enable3529()`）
- 配置某些 opcode 的行为或 gas
- 最后调用 `validate()` 做合法性检查

## validate
`validate` 函数确保：

1. 所有 256 个操作码都被设置（不能有 `nil`）
2. 如果设置了 `memorySize`，必须也设置了 `dynamicGas`

## `type operation struct`

代表一条 EVM 操作码的定义：

| 字段          | 类型              | 说明 |
|---------------|-------------------|------|
| `execute`     | `executionFunc`   | 实际执行函数 |
| `constantGas` | `uint64`          | 固定 gas 消耗 |
| `dynamicGas`  | `gasFunc`         | 动态 gas 计算函数（根据上下文） |
| `minStack`    | `int`             | 栈最小要求 |
| `maxStack`    | `int`             | 栈最大限制 |
| `memorySize`  | `memorySizeFunc`  | 内存大小需求 |
| `undefined`   | `bool`            | 是否为未定义操作码 |




