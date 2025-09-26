#todo #目录 
 
```ccard
type: folder_brief_live
```
 
crypto/crypto.go:PubkeyToAddress  [[accounts.keystore]]
crypto.Sign [[transaction_signing.go]]
crypto.Ecrecover  交易哈希+r,s,v -> 公钥

crypto  
┣ blake2b/  
┣ bn256/  
┣ ecies/  
┣ kzg4844/  
┣ secp256k1/  
┣ signify/  
┣ crypto.go  
┣ crypto_test.go  
┣ signature_cgo.go  
┣ signature_nocgo.go  
┗ signature_test.go


# todos
```go
// CreateAddress creates an ethereum address given the bytes and the nonce
func CreateAddress(b common.Address, nonce uint64) common.Address {
	data, _ := rlp.EncodeToBytes([]interface{}{b, nonce})
	return common.BytesToAddress(Keccak256(data)[12:])
}

```