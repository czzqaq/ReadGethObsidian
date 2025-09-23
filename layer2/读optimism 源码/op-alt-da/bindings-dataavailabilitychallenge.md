
# 说明
## abigen
这个代码由 [apigen](https://geth.ethereum.org/docs/tools/abigen) 自动生成。
apigen 依赖编译sol 产生的 abi。
> 以防我忘记：ABI 告诉外部，这个合约有哪些函数，参数是什么；Bin 是合约编译后的二进制。Bin + ABI，就等于看到了源码。其实可以这样想，.abi 和 .bin，就类似于.h 和 .o。

如果在生成文件时还带着 bin，那么abigen 中会多出一个 Deploy 方法。
```go
var DataAvailabilityChallengeMetaData = &bind.MetaData{

    ABI: "[{\"type\":\"constructor\",\"inputs\":[],\"stateMutability\":\"nonpayable\"},{\"type\":\"receive\",\"stateMutability\":\"payable\"},...",

    Bin: "0x60806040...",
}
```

```
# 编写一个 sol 文件
solc --abi --bin SimpleStorage.sol -o build

abigen --bin=build/SimpleStorage.bin \
       --abi=build/SimpleStorage.abi \
       --pkg=simplestorage \
       --out=simplestorage.go
```

### 调用

之后，这个合约可以在go 中使用。

下面是一段源代码中的例子。部署合约后，输入合约的地址，以及处理这个contract call transaction 的client，就可以把这个 当作单纯的类instance 使用。
实际上，这个 IsEcotone 是对 client 的rpc 调用，会阻塞直到获得结果。
```go
	l2Chain := sys.L2s()[chainIdx]
	l2Client, err := l2Chain.Nodes()[0].GethClient()
	gpoContract, err := bindings.NewGasPriceOracle(predeploys.GasPriceOracleAddr, l2Client)
	
	// 调用方法
	gpoEcotone, err := gpoContract.IsEcotone(&bind.CallOpts{BlockNumber: header.Number})
```

# DataAvailabilityChallenge 

见[specification](https://specs.optimism.io/experimental/alt-da.html#data-availability-challenge-contract)
