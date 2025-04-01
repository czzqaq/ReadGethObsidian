#archived 

# 程序内容
## signer
signer 其实是提供了一种具体的签名方法。它是transaction 的访问者visitor。

定义signer 接口：
```go
type Signer interface {
	// Sender returns the sender address of the transaction.
	Sender(tx *Transaction) (common.Address, error)

	// SignatureValues returns the raw R, S, V values corresponding to the
	// given signature.
	SignatureValues(tx *Transaction, sig []byte) (r, s, v *big.Int, err error)
	ChainID() *big.Int
	// Hash returns 'signature hash', i.e. the transaction hash that is signed by the private key. This hash does not uniquely identify the transaction.
	Hash(tx *Transaction) common.Hash
	// Equal returns true if the given signer is the same as the receiver.
	Equal(Signer) bool
}
```
具体的signer 实现跟随EIP 变化。

1. FrontierSigner
	1. genesis 到Homestead 分叉前
	2. 最基础的，没有chainid 。
2. HomesteadSigner
	1. 很类似FrontierSigner。
3. EIP155Signer
	1. EIP-155 激活后
	2. 引入链 ID 以防止跨链重放攻击.
4. EIP2930Signer
	1. Berlin 分叉后。
	2. 引入了txtype，type1
5. LondonSigner
	1. London 分叉后
	2. EIP-1559 类型交易（Type 2），支持 BaseFee 和 Tip
6. CancunSigner
	1. EIP-4844 Cancun 分叉后
	2. EIP-4844（Blob 交易，Type 3），签名器需要处理包含 blob 数据的新交易格式
7. PragueSigner
	1. 未来的 Prague 分叉


## 签名

```go
// SignTx signs the transaction using the given signer and private key.
func SignTx(tx *Transaction, s Signer, prv *ecdsa.PrivateKey) (*Transaction, error) {
	h := s.Hash(tx)
	sig, err := crypto.Sign(h[:], prv)
	if err != nil {
		return nil, err
	}
	return tx.WithSignature(s, sig)
}
```

1. 计算交易哈希，比如EIP2930Signer 以后的signer 都sign了chainid 和 parity字段。
2. 对 hash 进行密码学的椭圆曲线计算。返回的是一个[]byte ，作为下一步的入参 [[crypto]]
3. 调用modernSigner.SignatureValues。 这个函数也没做什么，单纯是把crypto 的输出解码，拿出来 r, s, parity 来。
4. 给交易的相关字段赋值，返回签名后的交易。

## 得到签名地址

```go
// Sender returns the address derived from the signature (V, R, S) using secp256k1
// elliptic curve and an error if it failed deriving or upon an incorrect
// signature.
//
// Sender may cache the address, allowing it to be used regardless of
// signing method. The cache is invalidated if the cached signer does
// not match the signer used in the current call.
func Sender(signer Signer, tx *Transaction) (common.Address, error) {
	if sigCache := tx.from.Load(); sigCache != nil {
		// If the signer used to derive from in a previous
		// call is not the same as used current, invalidate
		// the cache.
		if sigCache.signer.Equal(signer) {
			return sigCache.from, nil
		}
	}

	addr, err := signer.Sender(tx)
	if err != nil {
		return common.Address{}, err
	}
	tx.from.Store(&sigCache{signer: signer, from: addr})
	return addr, nil
}
```
1. 如果交易中已经储存了signer，就直接返回。
2. 如果没有，则计算sender 地址，并储存到 transaction中。（使用了atomic，保证线程安全）

其中，恢复sender 地址的核心函数：

```go
func recoverPlain(sighash common.Hash, R, S, Vb *big.Int, homestead bool) (common.Address, error) {
	if Vb.BitLen() > 8 {
		return common.Address{}, ErrInvalidSig
	}
	V := byte(Vb.Uint64() - 27)
	if !crypto.ValidateSignatureValues(V, R, S, homestead) {
		return common.Address{}, ErrInvalidSig
	}
	// encode the signature in uncompressed format
	r, s := R.Bytes(), S.Bytes()
	sig := make([]byte, crypto.SignatureLength)
	copy(sig[32-len(r):32], r)
	copy(sig[64-len(s):64], s)
	sig[64] = V
	// recover the public key from the signature
	pub, err := crypto.Ecrecover(sighash[:], sig)
	if err != nil {
		return common.Address{}, err
	}
	if len(pub) == 0 || pub[0] != 4 {
		return common.Address{}, errors.New("invalid public key")
	}
	var addr common.Address
	copy(addr[:], crypto.Keccak256(pub[1:])[12:])
	return addr, nil
}
```

可以发现，地址是pub 的后20位地址（排除掉32-12位），使用的恢复公钥的算法是：
crypto.Ecrecover。


# 相关
## 使用例
在[[blockchain.go-insertChain]] 中第一次提及：
```go
// in blockchain.go
// Start a parallel signature recovery (signer will fluke on fork transition, minimal perf loss)
SenderCacher().RecoverFromBlocks(
    types.MakeSigner(bc.chainConfig, chain[0].Number(), chain[0].Time()),
    chain
)
```


