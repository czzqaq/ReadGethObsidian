# 黄皮书

## 原理

这里对 [[4.4.1 Transaction Receipt#Bloom 过滤器计算（Bloom Filter Computation）]] 的内容做注释。


这里的Bloom 的bitmap 长度固定为 2048.
一共有三个哈希函数。set 的值是 
$$
h(x) \mod 2048
$$

我们需要的哈希值只要大于2048就行。8位的 byte 不够，16位的short 超了。
对 x 进行Keccak-256，从中提取3个short，分别位于第0,2,4位。黄皮书上说是大端，于是最高位是第0位。

## 记录内容
```go
// CreateBloom creates a bloom filter out of the give Receipt (+Logs)
func CreateBloom(receipt *Receipt) Bloom {
	var (
		bin Bloom
		buf = make([]byte, 6)
	)
	for _, log := range receipt.Logs {
		bin.add(log.Address.Bytes(), buf)
		for _, b := range log.Topics {
			bin.add(b[:], buf)
		}
	}
	return bin
}

```

注意到，被bloom 记录的，是产生日志的合约地址（主要的key，$O_a$ ），以及topic，且topic 的值（记录的就是哈希，见[[receipt.go#日志]]），直接就代表了  $Keccak-256(x)$
