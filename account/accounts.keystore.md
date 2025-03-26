本地的默认钱包，implement 了 Wallet 接口。


# 概念
#概念 

## Web3 Secret Storage

>Keys are stored as encrypted JSON files according to the Web3 Secret Storage specification.
// See https://github.com/ethereum/wiki/wiki/Web3-Secret-Storage-Definition for more information.

### Key Derivation Function

KDF 是一种函数，输入用户提供的密码+Salt，输出一个固定长度的密钥。使得密码不直接用于加密操作,因为起码，本地的电脑上不可能直接保存一个明文密码吧。
攻击者就算破解了电脑，拿到了一个KDF 数据，他也不能用，因为KDF 输入以后会过一个新的KDF，还是错误密码。当然，攻击者可以尝试，到底什么样的明文密码可以得到这个KDF。

作用：
- 添加salt，避免了相同密码产生相同密钥。
- 把密码转为固定长度，方便加密
- 提高安全性。

#### salt
添加salt 的主要目的的防止 [彩虹表攻击](https://zh.wikipedia.org/zh-cn/%E5%BD%A9%E8%99%B9%E8%A1%A8)
彩虹表攻击是指，攻击者自己维护了一个数据库，里面有了非常多预先计算好的KDF 结果-> 原密码，拿它到各个用户的KDF 中找，看看谁的在数据库中找到了，从而不是每个用户的破解都需要时间。
但是，通过salt，每个用户的KDF 生成都独一无二，彩虹表就没用了。

所以，salt 本身是可以明文储存的，它常常储存在配置文件里，和KDF 参数等一起保存。

#### 安全性
也就是，哪怕攻击者拿到了数据库里的KDF 密码，也很难找出来匹配的明文密码。
所以，这些密钥生成算法的时间复杂度都极其高，使得单次的试错尝试都很费时间，从而提高了安全性。比如PBKDF2：推荐迭代次数 `c` 至少为 100,000 或更高。
此外，为了防止GPU 加速，KDF 的内存需要非常高，从而限制了GPU 的计算能力。
KDF 算法的复杂度需要是密码长度、密码内容无关的，防止攻击者测量加密操作的时间而判断密码长度。


### 加密解密过程
#### 加密
1. 随机生成uuid，salt，iv。定义`ciper` = aes-128-ctr(ciperparams[iv])
2. 做DK = KDF(明文密码，salt)
3. DK[0..15] 是AES 的加密密钥。
4. 用`ciper` 加密用户的私钥，得到 `ciphertext`，记录 `ciphertext`。
5. 计算MAC，并记录。

$$
MAC = KECCAK(DK[16..31] ++ <ciphertext>)

$$

#### 解密

当接收到用户输入的密码时
1. 做DK = KDF(明文密码)
2. 先用MAC 做初次验证，如果和储存的MAC 不匹配，则cancel operation

$$
MAC = KECCAK(DK[16..31] ++ <ciphertext>)

$$
where, ++ is the concatenation operator; 

3. 使用DK[0..15] 作为密钥，解密 ciphertext 得到用户。


### Clef

Clef 是 Geth 提供的 **外部签名管理器**，作用是让私钥管理更加安全，避免在不可信环境中直接操作私钥。keystore 就是被Clef 使用的。具体见：[[signer]]

Clef 负责管理签名请求，而私钥是从keystore 中读取的。

### 格式

```json
{
    "crypto" : {
        "cipher" : "aes-128-ctr",
        "cipherparams" : {
            "iv" : "6087dab2f9fdbbfaddc31a909735c1e6"
        },
        "ciphertext" : "5318b4d5bcd28de64ee5559e671353e16f075ecae9f99c7a79a38af5f869aa46",
        "kdf" : "pbkdf2",
        "kdfparams" : {
            "c" : 262144,
            "dklen" : 32,
            "prf" : "hmac-sha256",
            "salt" : "ae3cd4e7013836a3df6bd7241b12db061dbe2c6785853cce422d148a624ce0bd"
        },
        "mac" : "517ead924a9d0dc3124507e3393d175ce3ff7c1e96529c6c555ce9e51205e9b2"
    },
    "id" : "3198bc9c-6672-5ab3-d995-4942343ae5b6",
    "version" : 3
}
```

如上，是一个version=3, 使用了pbkdf2 作为KDF 的配置文件。另一个KDF 是“scrypt”。
储存的内容都比较符合直觉。没什么好说的。



# 程序

## key.go

主要就是 Secret Storage 的实现。具体查看上面的概念部分即可。

```go
type Key struct {
    Id uuid.UUID // Version 4 "random" for unique id not derived from key data
    // to simplify lookups we also store the address
    Address common.Address
    // we only store privkey as pubkey/address can be derived from it
    // privkey in this struct is always in plaintext
    PrivateKey *ecdsa.PrivateKey
}
type CryptoJSON struct {
    Cipher       string                 `json:"cipher"`
    CipherText   string                 `json:"ciphertext"`
    CipherParams cipherparamsJSON       `json:"cipherparams"`
    KDF          string                 `json:"kdf"`
    KDFParams    map[string]interface{} `json:"kdfparams"`
    MAC          string                 `json:"mac"`
}
type encryptedKeyJSONV3 struct {
    Address string     `json:"address"`
    Crypto  CryptoJSON `json:"crypto"`
    Id      string     `json:"id"`
    Version int        `json:"version"`
}
```

提供了一个Key 的储存接口，在[[#keystore.go]] 中被使用了，详见那里。
## keystore.go
```go
type KeyStore struct {
    storage  keyStore                     // Storage backend, might be cleartext or encrypted
    cache    *accountCache                // In-memory account cache over the filesystem storage
    changes  chan struct{}                // Channel receiving change notifications from the cache
    unlocked map[common.Address]*unlocked // Currently unlocked account (decrypted private keys)
    wallets     []accounts.Wallet       // Wallet wrappers around the individual key files
    updateFeed  event.Feed              // Event feed to notify wallet additions/removals
    updateScope event.SubscriptionScope // Subscription scope tracking current live listeners
    updating    bool                    // Whether the event notification loop is running
    mu       sync.RWMutex
    importMu sync.Mutex // Import Mutex locks the import to prevent two insertions from racing
}
```

### NewAccount
储存一个 account 的过程：
NewAccount（password）
    1. storeNewKey(在)
        1. newKey :使用ECDSA 生成一个ecdsa.PrivateKey，然后储存在 `Key` （如上）中，其中address                 是公钥派生的address，详见：[[crypto]]
        2. 创建了一个账号 `var account`。见：[[account.go_type_Account]]
        3. ks.StoreKey  // 在[[#plain.go]] 中被定义，储存文件。
    2. ks.cache.add(account) 其实可以认为wallet 就存的是account 和 private key。反正它的关键就是对account 的管理对吧。
    3. ks.refreshWallets()
           在ks 中，有一个account 队列，有一个wallet 队列，需要让account 和 wallet 中包的account 完全对应，方法是看着account 队列（ks.cache)，调整wallet 队列。
           写得其实挺麻烦，因为要释放 Event消息，来说明哪些钱包队列中的内容被drop了（WalletDropped），新增了哪些钱包（WalletArrived）。逻辑是对每个wallet 队列上最开头的元素，和全部的account 内容做比较。
           举个例子：URL1 URL2 对 URL1 ，函数的运行方法是，先把URL1 的wallet 加上（账户相同），钱包队列的URL1 被移走，然后URL2 添加，因为队列上没有了。
           第二个例子：URL2, URL1 对 URL1。URL2 和 钱包的第一个元素URL1 比，大了，所以drop 掉URL1。之后添加URL2 和 URL1。生成的事件是URL1 drop，URL2添加，URL1 添加。
           之所以写这么麻烦，主要还是因为wallets 序列还会受到外界的控制。


### export，import
作用详见：[[signer]]
原理详见：[[accounts.keystore#passphrase.go]]

主要功能如下
**Export** 把一个Account导出（Export）为加密（Encrypt）后的文本储存为json，输入的passphrase 是验证的密码（比如123456password 这种用户定义的密码，代码中称为auth）
Import 反过来，JSON->Account。
```go
func (ks *KeyStore) Export(a accounts.Account, passphrase, newPassphrase string) (keyJSON []byte, err error) {
func (ks *KeyStore) Import(keyJSON []byte, passphrase, newPassphrase string) (accounts.Account, error) {
```

第二组：
把输入的ECDSA 私钥，保存为二进制数据，产生一个key 对象，输出为account。就，本来account 就是一个key（包括了公钥，和公钥一个意思的地址，以及私钥，这三个跟ECDSA有关的，加上一个URL，来指明相关的资源）
```go
func (ks *KeyStore) ImportECDSA(priv *ecdsa.PrivateKey, passphrase string) (accounts.Account, error)
// 没有对应的Export
```

第三组：
```go
// ImportPreSaleKey decrypts the given Ethereum presale wallet and stores

// a key file in the key directory. The key file is encrypted with the same passphrase.

func (ks *KeyStore) ImportPreSaleKey(keyJSON []byte, passphrase string) (accounts.Account, error)
```

详见[[#presale.go]]
### as Backend

再次回顾[[account.go]] 中声明的backend：

```
type Backend interface {

    // Wallets retrieves the list of wallets the backend is currently aware of.
    // The resulting wallet list will be sorted alphabetically based on its internal
    // URL assigned by the backend. Since wallets (especially hardware) may come and go, the same wallet might appear at a different positions in the list during
    // subsequent retrievals.

    Wallets() []Wallet
    // Subscribe creates an async subscription to receive notifications when the
    // backend detects the arrival or departure of a wallet.
    Subscribe(sink chan<- WalletEvent) event.Subscription
}
```

关于subscription本身，见：

[[subscription.go]]


## plain.go

实现了一种keystore（这是个Interface），它只是简单的把配置文件按照标准格式写入file。


## passphrase.go


## presale.go



## subscriber.go


# go 语法

## Mutex

keystore 中使用了它
```go
mu sync.RWMutex  
importMu sync.Mutex // Import Mutex locks the import to prevent two insertions from racing
```
**Mutex** 普通的互斥锁
- mutex.lock() 调用后，其他的routine 再调用就会被阻塞。
- mtex.unlock()

**RWMutex**
- RLock()   允许多个 Goroutine 同时获取读锁，但如果写锁被占用，则会阻塞。
- RUnlock()
- `Lock()`：加写锁。只有当前没有任何读锁或写锁时，才能获取写锁。
- Unlock()

