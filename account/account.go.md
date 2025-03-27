#archived 

# 说明
提供了wallet、account、backend 的抽象定义。

# Go 语法

## type 和 接口

如下是一个简单的实践：
```go
type Speaker interface {
	Speak() string
}

type Dog struct{}
func (d Dog) Speak() string { // 直接声明接口就好
	return "Woof!"
}

func main() {
	var s Speaker = Dog{}
	fmt.Println(s.Speak()) // 输出: Woof!
}
```
- 接口是隐藏实现的，不需要implement关键字 
- 一个type定义方法，同样的过程可以定义接口
- interface 本身也可以作为type，就好像`Wallet` 接口。
- 在 Go 语言中，**接口是隐式实现的**. 一个type 拥有了接口里的全部方法，它就是这个接口的类型，同时，也 **必须实现接口的所有方法**，否则无法赋值给接口变量。

### 类型断言
```go
type Speaker interface {
	Speak() string
}

type AnotherInterface interface {
     Boo()
}

type Dog struct{}
func (d Dog) Speak() string {
	return "Woof!"
}

func (d Dog) Boo() {
	fmt.Println("Boo! ")
}

func main() {
	var s Speaker = Dog{}
	s.Boo()
}
```

这段代码是不行的。因为Boo() 虽然确实是Dog的方法，但是编译器不会做这种检查。speaker确实就是没有Boo 方法。

可以这样写：
```go
func main() {
	var s Speaker = Dog{}
	
	// 尝试将 s 转换为 AnotherInterface
	if a, ok := s.(AnotherInterface); ok {
		a.Boo() // ✅ 现在可以调用 Boo()
	} else {
		fmt.Println("s does not implement AnotherInterface")
	}
}
```


###  接口继承
```go
package main

import "fmt"

type Speaker interface {
	Speak() string
	AnotherInterface // 继承 Boo() 方法
}

type AnotherInterface interface {
	Boo()
}

type Dog struct{}

func (d Dog) Speak() string {
	return "Woof!"
}

func (d Dog) Boo() {
	fmt.Println("Boo!")
}

func main() {
	var s Speaker = Dog{} // Dog 现在完全实现了 Speaker
	s.Boo()  // ✅ 现在可以调用 Boo()
	s.Speak() // ✅ 也可以调用 Speak()
}
```

### 重复命名

下面的代码不会报错

```go

// file1.go
type Interface1 interface {
	Boo() // ✅ 这个 Boo 和 Interface2 里的 Boo 没有关系
}

// file2.go
type Interface2 interface {
	Boo() // ✅ 这个 Boo 和 Interface1 里的 Boo 也是独立的
}

type Interface3 interface {
	Interface1
	Interface2
}
```

如果一个 `struct` **实现了 `Boo()` 方法**，它可以**同时满足 `Interface1` 和 `Interface2`**：
```go
type MyStruct struct{}

func (m MyStruct) Boo() {
	fmt.Println("Boo!")
}

func main() {
	var i1 Interface1 = MyStruct{} // ✅ MyStruct 实现了 Interface1
	var i2 Interface2 = MyStruct{} // ✅ MyStruct 也实现了 Interface2

	i1.Boo() // 输出: Boo!
	i2.Boo() // 输出: Boo!
}
```


### “显式”声明接口实现


```go
type Speaker interface {
	Speak() string
}

type Dog struct{}

// Dog 实现了 Speaker 接口
func (d Dog) Speak() string {
	return "Woof!"
}

// 编译时检查 Dog 是否实现了 Speaker
var _ Speaker = (*Dog)(nil) // ✅ 如果 Dog 没有实现 Speaker，编译会报错
```


另一种方式是**在结构体里嵌入接口**，让 IDE 更容易识别：
```go
type Speaker interface {
	Speak() string
}

type Dog struct {
	Speaker // 这里嵌入接口
}

func (d Dog) Speak() string {
	return "Woof!"
}
```

在vscode 中，右键接口名，view all implementations，就可以看出来。
![[Pasted image 20250320085907.png]]
此外，通过命名也可以：
```go
type Wallet interface {
}

type Wallet struct{}

```

## "枚举"
**Go 语言没有 `enum` 关键字**，但可以使用 **`const` + `iota`** 来实现类似 **枚举（enum）** 的功能。
const 是编译时获取，所以这种枚举不占内存，效率高。

```go
type WalletEventType int // 定义一个新的类型

const (
	WalletArrived WalletEventType = iota // 0
	WalletOpened                         // 1
	WalletDropped                        // 2
)

func main() {
	event := WalletOpened

	// 检查枚举值
	switch event {
	case WalletArrived:
		fmt.Println("A new wallet has arrived.")
	case WalletOpened:
		fmt.Println("Wallet is opened.")
	case WalletDropped:
		fmt.Println("Wallet has been removed.")
	}
}
```

此外还有：
```go
const (
	FlagRead  = 1 << iota // 1 << 0 = 1
	FlagWrite             // 1 << 1 = 2
	FlagExecute           // 1 << 2 = 4
)

const (
	_       = iota // 跳过 0
	One            // 1
	Two            // 2
)

const (
	First  = iota + 1 // 1
	Second            // 2
	Third             // 3
)
```
# 内容
定义了
```go

type Account struct {
    Address common.Address `json:"address"` // Ethereum account address derived from the key
    URL     URL            `json:"url"`     // Optional resource locator within a backend
}

type Wallet interface{}
type Backend interface{}
type WalletEventType int

type WalletEvent struct {
    Wallet Wallet          // Wallet instance arrived or departed
    Kind   WalletEventType // Event type that happened in the system
}

```

## Wallet
在几个文件夹里implement 了不同的接口：
[[accounts.scwallet]]
[[accounts.usbwallet]]
[[accounts.keystore]]

这3个文件夹在不同的package 下，定义了不同的Wallet struct，可以按需使用

```go
type Wallet struct {
    ...
}
```
这个写法很神奇，Wallet 对外表现就是接口一样。

因为接口的实现是隐式的，所以这里可读性很差。(不过好处是可以解耦，毕竟go 没有类，一个type 的方法实现可以在多个文件下。)
geth 中使用的方法，就是把结构体的type 命名得和接口一样。


| **钱包类型**         | **存储方式**    | **安全性** | **适用场景**  |
| ---------------- | ----------- | ------- | --------- |
| `scwallet`       | 智能卡         | 高       | 企业级安全     |
| `keystoreWallet` | Keystore 文件 | 中       | 个人用户，软件钱包 |
| `usbwallet`      | 硬件钱包（USB）   | 高       | 硬件钱包用户    |

此外，还有一个不同的，`type ExternalSigner struct` 见：
[[accounts.external]]

## Backend
同样的，上述三个文件夹位置上，也实现了Backend。这个接口比较简单，主要就是用来做一个事件的生成器。

```go
// Backend is a "wallet provider" that may contain a batch of accounts they can

// sign transactions with and upon request, do so.

type Backend interface {

    // Wallets retrieves the list of wallets the backend is currently aware of.
    // The returned wallets are not opened by default. For software HD wallets this
    // means that no base seeds are decrypted, and for hardware wallets that no actual
    // connection is established.
    //
    // The resulting wallet list will be sorted alphabetically based on its internal
    // URL assigned by the backend. Since wallets (especially hardware) may come and
    // go, the same wallet might appear at a different positions in the list during
    // subsequent retrievals.
    Wallets() []Wallet

    // Subscribe creates an async subscription to receive notifications when the
    // backend detects the arrival or departure of a wallet.
    Subscribe(sink chan<- WalletEvent) event.Subscription

}
```

例如用例：
```go
    am := makeAccountManager(ctx)
    backends := am.Backends(keystore.KeyStoreType)
    ks := backends[0].(*keystore.KeyStore) // KeyStore 就是具体的这个实现了backend 接口的instance
    ks.Update(account, password, newPassword) // Update 是一个ks 自己独有的方法，
```

详见：
[[manager.go]]

## account
[[account.go_type_Account]]