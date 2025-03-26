
# 介绍
指：[[Hierarchical Deterministic Wallet]]

提供了`DefaultBaseDerivationPath`：
```go
type DerivationPath []uint32

var DefaultBaseDerivationPath = DerivationPath{0x80000000 + 44, 0x80000000 + 60, 0x80000000 + 0, 0, 0}
```

# 函数
## ParseDerivationPath
ParseDerivationPath: 把字符串转为`DerivationPath`

```go
bigval, ok := new(big.Int).SetString(component, 0)
```

```go
func (z *Int) SetString(s string, base int) (*Int, bool)
```

- `s string`：要解析的字符串，即 `component`。
- `base int`：进制，`0` 表示**自动检测进制**：
    - 以 `0x` 或 `0X` 开头 → 16 进制
    - 以 `0` 开头（但不是 `0` 本身）→ 8 进制
    - 其他情况 → 10 进制

对于每个component，去掉开头可能的`'` （hardened paths）


使用big.Int 的库，使得可以处理各类输入，负数、很大的数字等，之后再对数字做检查。
后续有范围检查。检查基准：
 **普通索引**（non-hardened）：`0` 到 `2^31 - 1`（即 `0x00000000` 到 `0x7FFFFFFF`）。
**硬化索引**（hardened）：`2^31` 到 `2^32 - 1`（即 `0x80000000` 到 `0xFFFFFFFF`）。

## marshalJSON

```go
// MarshalJSON turns a derivation path into its json-serialized string

func (path DerivationPath) MarshalJSON() ([]byte, error) {

    return json.Marshal(path.String())

}
```

定义了一个类型DerivationPath 的方法。
如果DerivationPath 有MarshalJSON 的字段，它就会在调用`json.Marshal` 时，输出自定义格式。



```go
type Person struct {
	Name string
	Age  int
}
```

默认情况下，`json.Marshal` 会返回：
```json
{"Name":"Alice","Age":30}

```

```go
func (p Person) MarshalJSON() ([]byte, error) {
	return json.Marshal(map[string]interface{}{
		"name haha I can define a new format": p.Name,
	})
}
```
序列化后结果：
```go
{"name haha I can define a new format":"Alice"}
```
