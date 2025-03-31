
# 内容
在 [[accounts.keystore]] 、等中提到


这个文件是一个事件订阅系统的实现，主要提供以下功能：

1. **基础订阅机制**：通过 `Subscription` 接口定义了订阅的基础功能。
2. **自动重试**：通过 `Resubscribe` 和 `ResubscribeErr` 实现了订阅失败后的自动重试逻辑。
3. **批量管理**：通过 `SubscriptionScope` 提供了批量管理多个订阅的能力。

这些功能在区块链客户端中非常重要，因为链上事件监听可能会因为网络波动等原因中断，需要自动恢复或手动集中管理订阅。

[发布-订阅模式（Publish-Subscribe Pattern）](https://juejin.cn/post/6844903834196656141)


## SubscriptionScope

它提供了管理多个subscription的能力，本身实现异常简单，不过要理解这个概念。

```go
// Track starts tracking a subscription. If the scope is closed, Track returns nil. The
// returned subscription is a wrapper. Unsubscribing the wrapper removes it from the
// scope.
func (sc *SubscriptionScope) Track(s Subscription) Subscription 

// Close calls Unsubscribe on all tracked subscriptions and prevents further additions to
// the tracked set. Calls to Track after Close return nil.
func (sc *SubscriptionScope) Close()
// Count returns the number of tracked subscriptions.
// It is meant to be used for debugging.
func (sc *SubscriptionScope) Count() int
```

## Resubscriber

- 它是一个subscriber，这个subscriber 的type是 `resubscribeSub`
- 但是，另外有一个进程，每当subscribe 错误的时候，就重新建立链接。建立连接的策略遵循[backoff pattern](https://zhuanlan.zhihu.com/p/619872368)
- 调用Unsubscribe，订阅会被安全终止。

这里的逻辑跟区块链完全无关，不过确实是对go 机制和多进程处理的一次很好的实践。

具体逻辑看代码吧。
入口：
`Resubscribe`，和 `ResubscribeErr`，两者逻辑相同，区别只有`ResubscribeErr` 多增加了一个对于Error 的处理。

```go
func ResubscribeErr(backoffMax time.Duration, fn ResubscribeErrFunc) Subscription {
    s := &resubscribeSub{
        waitTime:   backoffMax / 10,
        backoffMax: backoffMax,
        fn:         fn, // 具体的运行时函数，subscribe 到底subscribe 了什么
        err:        make(chan error),
        unsub:      make(chan struct{}, 1),
    }
    go s.loop() // 一个维持resubs 的线程
    return s // 初始化s 这个实例并返回
}

func (s *resubscribeSub) loop() {
    defer close(s.err)
    var done bool
    for !done {
        sub := s.subscribe()  // 一个无限循环函数，除非遇到err，否则不会退出
        if sub == nil {
            break
        }
        done = s.waitForError(sub) // 如果是因为err，那么done 是true（等到了error）
        sub.Unsubscribe()
    }

}
```

以下关注：
1. 一个通用的检测键盘输入和中断的流程。
2. 非阻塞式 chan 的使用

```go
func (s *resubscribeSub) subscribe() Subscription {
    subscribed := make(chan error)
    var sub Subscription

    for {
        s.lastTry = mclock.Now()
        ctx, cancel := context.WithCancel(context.Background())
        go func() {
            rsub, err := s.fn(ctx, s.lastSubErr)
            sub = rsub
            subscribed <- err
        }()
        select { // 对传入的fn 的返回值的处理，非阻塞式（也不能是阻塞式，不然直接写成一个函数就行了
        case err := <-subscribed:
            cancel()
            if err == nil {
                if sub == nil {
                    panic("event: ResubscribeFunc returned nil subscription and no error")
                }
                return sub
            }

            // Subscribing failed, wait before launching the next try.
            if s.backoffWait() {
                return nil // unsubscribed during wait
            }
        case <-s.unsub:
            cancel()
            <-subscribed // avoid leaking the s.fn goroutine.
            return nil
        }
    }
}
```


# 语法

```go
func NewSubscription(producer func(<-chan struct{}) error) Subscription {
    s := &funcSub{unsub: make(chan struct{}), err: make(chan error, 1)}
    go func() {
        defer close(s.err)
        err := producer(s.unsub)
        s.mu.Lock()
        defer s.mu.Unlock()
        if !s.unsubscribed {
            if err != nil {
                s.err <- err
            }
            s.unsubscribed = true
        }
    }()
    return s
}
```
## chan 
`go` 关键字用于启动一个轻量级线程（Goroutine）
`chan` 是 Go 中的并发通信工具，用于在 Goroutine 之间传递数据。

一个简单的例子：
```go
func main() {
    ch := make(chan int) // 创建一个通道

    // 启动一个 Goroutine，向通道发送数据
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i // 向通道发送数据
        }
        close(ch) // 关闭通道
    }()

    // 从通道接收数据，直到通道关闭
    for val := range ch {
        fmt.Println(val)
        time.Sleep(1 * time.Second) // 每次接收后等待 1 秒
    }
}
```

### 阻塞

Go 的通道是**阻塞式的**，这意味着：

1. 向通道发送数据时（`ch <- value`）：
    
    - 如果没有 Goroutine 正在从通道接收数据（`<-ch`），发送操作会阻塞，直到有 Goroutine 开始接收数据。
    - 如果通道已关闭，发送操作会引发运行时错误。
2. 从通道接收数据时（`<-ch`）：
    
    - 如果通道中没有数据可供接收，接收操作会阻塞，直到有数据写入通道。
    - 如果通道已关闭且数据已被全部接收完毕，接收操作会立即返回通道的零值。

例如，上面的例子里，每隔1秒输出一个数字，从0 println 到4，主程序一共运行5秒。Goroutine 的存活时间是4秒。


接收也是阻塞的，比如：
```
func main() {
    ch := make(chan int) // 创建一个无缓冲通道

    // 启动一个 Goroutine，向通道发送数据
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i // 向通道发送数据
            time.Sleep(1 * time.Second) // 每次发送后等待 1 秒
        }
        close(ch) // 关闭通道
    }()

    // 从通道接收数据，直到通道关闭
    for val := range ch {
        fmt.Println(val) // 打印接收到的值
    }
}
```

这个程序的行为是每秒输出一个值，总运行时间5秒。


使用select 可以实现非阻塞接受

```go
func main() {
    ch := make(chan int)

    go func() {
        for i := 0; i < 5; i++ {
            ch <- i // 向通道发送数据
            time.Sleep(1 * time.Second)
        }
        close(ch)
    }()

    for {
        select {
        case val, ok := <-ch:
            if !ok {
                fmt.Println("Channel closed")
                return
            }
            fmt.Println("Received:", val)
        default:
            fmt.Println("No data, doing other work")
            time.Sleep(500 * time.Millisecond) // 模拟其他工作
        }
    }
}
```
### chan struct{}
表示一个传递空结构体的通道。空结构体 `struct{}` 是零字节的，常用于只发送信号而不传递数据。例如：
```go
ch := make(chan struct{})
go func() {
    // do many works.
    ch <- struct{}{} // 向通道发送一个信号
}()
<-ch // 接收信号
fmt.Println("Received signal, the subroutine work done")
```

> [!caution] 不阻塞还有通道关闭的情况
> 也就是，不能简单理解为一直阻塞直到得到信号，通道被关闭后也会“得到信号”


#### 实现一个轻量的锁
```go
f.sendLock = make(chan struct{}, 1)
f.sendLock <- struct{}{} // 初始化时放入一个 “令牌”

```

这个 channel 被用作一个 **轻量级的互斥锁（mutex）**，类似于信号量或令牌桶。
使用方式：
```go
<-f.sendLock // 获取“锁”，阻塞直到成功取到
// 临界区代码
fmt.Print("helloworld")
f.sendLock <- struct{}{} // 释放“锁”
```

这样，就可以保证永远最多只有一个Routine 可以拿到这个令牌，使用资源。这个人用完资源后，必须把令牌放回去。

### 只读通道和只写通道
```go
package main

import "fmt"

func producer(ch chan<- int) { // 只写通道
	for i := 0; i < 5; i++ {
		ch <- i // 发送数据
	}
	close(ch) // 关闭通道
}

func consumer(ch <-chan int) { // 只读通道
	for val := range ch { // 从通道接收数据
		fmt.Println(val)
	}
}

func main() {
	ch := make(chan int)
	go producer(ch) // 启动生产者
	consumer(ch)    // 消费者接收数据
}
```


就是一个契约式写法，只读通道和只写通道也是通道，你完全可以写成：

```go
func producer(ch chan int)
func consumer(ch chan int)
```
### 通道关闭

- **向已关闭的通道写入数据会引发 `panic`**。
- **从已关闭的通道读取数据不会 `panic`**，而是会返回通道类型的零值。
- **一个通道只能被关闭一次，重复关闭会引发 `panic`**。

这样关闭通道：
```go
defer close(s.err)
```
 关闭通道后，所有consumer 上的阻塞会被解除。

### 多线程

Go 的通道本身是线程安全的，可以支持多个生产者和多个消费者同时操作，不需要额外的锁。
- 注意，只有一个producer  可以关闭通道，否则会panic。
- 会有不均等消费的问题。


## 匿名函数
```go

func Resubscribe(backoffMax time.Duration, fn ResubscribeFunc) Subscription {
    return ResubscribeErr(backoffMax, func(ctx context.Context, _ error) (Subscription, error) {
        return fn(ctx)
    })
}
```

如上，`func(inputs)(outputs) {// execusions}` 定义了一个匿名函数，注意大括号被包在括号里了，这个写法在js 里也特别多。

