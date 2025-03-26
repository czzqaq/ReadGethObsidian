#目录


```ccard
type: folder_brief_live
```



在 [[cmd.geth]] 中使用了：

`AccountManager` 
`WalletEvent` 
`Wallet` [[account.go]]
`Wallet.SelfDerive` 
`hd`  [[hd.go]]



account 就是一个key（包括了公钥，和公钥一个意思的地址，以及私钥，这三个跟ECDSA有关的，加上一个URL，来指明相关的资源）
这里主要定义了account 的导入和导出，不同的导入和导出方式产生了不一样的钱包，比如本地geth 原生的本地储存 [[accounts.keystore]]；


preliminary ：
为了更好的理解account 中的核心密钥，需要先理解[[crypto]]

