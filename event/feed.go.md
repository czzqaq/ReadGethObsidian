这个文件实现了一个 **事件分发系统**，用于在 Golang 中实现 **一对多的广播通信机制**。它允许你将一个事件（数据）通过一个 `channel` 广播给多个订阅者。

这个机制在构建 **发布-订阅（Pub/Sub）模型** 时非常有用.

# 代码
## 工作流程

### 1. **订阅（Subscribe）**

```go
func (f *Feed) Subscribe(channel interface{}) Subscription
```
- 检查 `channel` 是可发送的。
- 设置 `etype`（第一次调用时）为订阅的 `channel` 的元素类型。
- 将新的 `channel` 放入 `inbox`，等待下次 `Send` 时处理。

---

### 2. **发送（Send）**

```go
func (f *Feed) Send(value interface{}) (nsent int)
```

- 检查发送的 `value` 类型是否与订阅的 `channel` 类型一致。
- 将 `inbox` 中的 channel 合并到 `sendCases` 中。
- 尝试向每个订阅者发送事件：
    - 优先尝试非阻塞发送（TrySend）
    - 如果阻塞则进入 `reflect.Select` 等待某些订阅者可接受数据
- 如果有订阅者已取消，会收到 `removeSub` 消息并清理其 `channel`
- 发送完毕后释放发送锁

---

### 3. **取消订阅（Unsubscribe）**

```go
func (sub *feedSub) Unsubscribe()
```

- 将订阅者从 `inbox` 或 `sendCases` 删除
- 如果当前没有在执行 `Send`，会立即删除
- 否则通过 `removeSub` 通知正在执行的 `Send` 来清理


# 使用
## 简单用例
```go
var f event.Feed

ch1 := make(chan string, 10)
ch2 := make(chan string, 10)

f.Subscribe(ch1)
f.Subscribe(ch2)

f.Send("hello subscribers")

fmt.Println(<-ch1) // "hello subscribers"
fmt.Println(<-ch2) // "hello subscribers"
```

# 语法
## sync.Once
用于 **确保某个操作只执行一次**，即使在多个 goroutine 中并发调用，也只会执行一次。适用于初始化操作、懒加载、单例模式等需要“只执行一次”的场景。

例：
```go
package main

import (
	"fmt"
	"sync"
)

var once sync.Once

func initOnce() {
	fmt.Println("This will only print once.")
}

func main() {
	for i := 0; i < 5; i++ {
		go func() {
			once.Do(initOnce)
		}()
	}

	// 等待 goroutines 执行（仅为演示）
	time.Sleep(1 * time.Second)
}
```

在 feed.go 中：
```go
// in Subscribe()
f.once.Do(func() { f.init(chantyp.Elem()) })
```

从而使得对于操作type 的检查只进行一次。

## reflect 实现通用channel
为了实现一个可以传输任何数据类型的channel，使用了reflect 库相关的机制。

**反射** 是一种程序在运行时检查和操作自身结构的能力。在 Go 中，反射主要用于：

- **获取变量的类型和值**
- **修改变量的值**
- **调用变量的方法**
- **动态操作结构体字段**

### 通用例子
```go
package main

import (
	"fmt"
	"reflect"
)

func add(a, b interface{}) interface{} {
	va := reflect.ValueOf(a)
	vb := reflect.ValueOf(b)

	// 类型检查
	if va.Kind() != vb.Kind() {
		panic("add: mismatched types")
	}

	switch va.Kind() {
	case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
		return va.Int() + vb.Int()

	case reflect.Uint, reflect.Uint8, reflect.Uint16, reflect.Uint32, reflect.Uint64:
		return va.Uint() + vb.Uint()

	case reflect.Float32, reflect.Float64:
		return va.Float() + vb.Float()

	case reflect.String:
		return va.String() + vb.String()

	default:
		panic("add: unsupported type " + va.Kind().String())
	}
}
```
### interface {}
```go
interface{}
```
表示一个 **不包含任何方法的接口类型**。由于所有类型都至少实现了零个方法，因此 **所有类型都可以赋值给 `interface{}`**。ValueOf 把一个具体的值映射为通用的`Value`类型。

### reflect.Select
它允许你在运行时构建一个 `select` 语句，**动态选择任意数量的通道进行发送或接收操作**。
例如
```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	// 创建多个通道
	ch1 := make(chan string, 1)
	ch2 := make(chan string, 1)
	ch3 := make(chan string, 1)

	msg := "hello"

	// 构建 SelectCase 列表
	cases := []reflect.SelectCase{
		{
			Dir:  reflect.SelectSend,
			Chan: reflect.ValueOf(ch1),
			Send: reflect.ValueOf(msg),
		},
		{
			Dir:  reflect.SelectSend,
			Chan: reflect.ValueOf(ch2),
			Send: reflect.ValueOf(msg),
		},
		{
			Dir:  reflect.SelectSend,
			Chan: reflect.ValueOf(ch3),
			Send: reflect.ValueOf(msg),
		},
	}
	// 广播消息：每次 select 会选择一个通道成功发送
	for len(cases) > 0 {
		chosen, _, _ := reflect.Select(cases)
		fmt.Printf("Sent to channel %d\n", chosen)

		// 发送成功后移除对应的 case，防止重复发送
		cases = append(cases[:chosen], cases[chosen+1:]...)
	}

}
```
它等价于：
```go
select {
case ch1 <- msg:
case ch2 <- msg:
case ch3 <- msg:
}
```

### 在feed.go中

在 feed.go 中：
```go
func (f *Feed) Subscribe(channel interface{}) Subscription {
	chanval := reflect.ValueOf(channel)      // 把 channel 包装成 reflect.Value
	chantyp := chanval.Type()                // 获取 channel 的类型
	f.once.Do(func() { f.init(chantyp.Elem()) }) // 初始化 Feed，记录通道元素类型
```

```go
func (f *Feed) Send(value interface{}) (nsent int) {
	rvalue := reflect.ValueOf(value)  // 把任意值变成 reflect.Value
	if f.etype != rvalue.Type() {
		panic(...) // 确保类型一致
	}

	// 将 rvalue 设置为每个已注册 case 的发送值
	for i := firstSubSendCase; i < len(f.sendCases); i++ {
		f.sendCases[i].Send = rvalue
	}
```

