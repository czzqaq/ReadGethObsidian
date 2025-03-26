# 说明
整个功能的入口。

从它的开始说明的包：`package main` 能看出来。
当用户输入 geth 时，就进入了这里。


阅读时，参考了：
https://learnblockchain.cn/article/3912

# go 语法
Go 的 `init()` 函数是一个特殊的函数，它会在程序启动时自动执行（在 `main()` 执行之前）。每个包都可以定义一个或多个 `init()` 函数，这些函数会按照包的初始化顺序依次运行。

定义了一个全局的`app`变量，然后在`init()` 中初始化，最后在main() 中运行起来。

init() 函数是迭代运行的，每个import 都会调用一次init()
而main() 函数是全应用程序唯一。

定义了一套命令行，借用：[这个开源的命令行框架](https://github.com/urfave/cli)

> [!tip] 这是常用的创建cmd的方法
> 

在init 中，说明了
```
cmd.Action = geth
```
也就是，处理cmd 的主力就是`geth` 函数了。


整个app 的定义在这个框架下，具体见：[[urfave_cli]]



# geth


它里面调用了（如下标题）

## prepare

检查和决定当前所在的网络，比如是sepolia 吗


## makeFullNode
一个通用的上下文加载器，它会返回`stack`，也就是一个node

关于node，详见 [[node]]


## startNode
### 语法
#### go func
`go` + 函数，启动一个线程
`func() {} ` 声明一个匿名函数
`func()[]()` 调用它

#### defer


1. `defer` 语句会立即被注册，但不会立即执行。
2. 在函数运行结束后，进行它们
3. 注册顺序是 **先注册的最后执行**（类似栈的后进先出原则）。

例如：
``` go
file, err := os.Open("example.txt")
if err != nil {
    log.Fatal(err)
}
defer file.Close() // 确保文件在函数结束时被关闭
```


### 钱包和账户
#### 相关代码

``` go
    events := make(chan accounts.WalletEvent, 16)

    stack.AccountManager().Subscribe(events)
   for event := range events {
            switch event.Kind {
            case accounts.WalletOpened:
                ...
                event.Wallet.SelfDerive(derivationPaths, ethClient)
           ...
   ...
   
```

#### 语法
##### make
`make` 是 Go 内置的一个函数，用来创建并初始化以下三种类型的复合数据结构：

- **切片（slice）**
- **映射（map）**
- **通道（channel）**

##### chan

`chan` 是 Go 中的关键字，用来定义和操作通道。通道是一种类型安全的通信机制，允许多个 goroutine 在不需要显式锁的情况下安全地交换数据。
``` go
var channelName chan Type
```
声明一个channel
通道可以是**阻塞式的**，也可以是**带缓冲的**
（`make(chan T, capacity)`）定义一个带buffer 的event channel

更加复杂的chan 类型，见：
[[subscription.go#chan 和 Goroutine]]

#### account 
`AccountManager`
`WalletEvent`
`Wallet`
`Wallet.SelfDerive`

见 [[account]]

#### DerivationPath

> `DerivationPath` 表示一种用于从 **分层确定性钱包**（Hierarchical Deterministic Wallet, HD Wallet）中派生特定账户地址的路径。它是一个**计算机友好的格式**，通常用来标识 HD 钱包的某个具体账户或地址。

HD 钱包使用 **BIP-32**（比特币改进提案）标准来生成一棵加密密钥树。
BIP-44 是 BIP-32 的扩展，统一了派生路径的规则。
SLIP-44 把type 分给了Ethereum

详见：
[[Hierarchical Deterministic Wallet]]

`Ledger` 的URL scheme ，指在用Ledger 与钱包软件交互时，只支持一个账户。ledger 是一个本地密钥管理软件。


### sync 事件

起了一个线程，来处理`downloader.DoneEvent{}`

应该是比较独立的，暂时不看了。

### 核心

``` go
rpcClient := stack.Attach()
ethClient := ethclient.NewClient(rpcClient)

events := make(chan accounts.WalletEvent, 16)
stack.AccountManager().Subscribe(events)

for event := range events {
   case accounts.WalletOpened:
      event.Wallet.SelfDerive(derivationPaths, ethClient)
      
```
[[rpc]]

例如：
```bash
geth --http --http.addr "127.0.0.1" --http.port "8545"
```

