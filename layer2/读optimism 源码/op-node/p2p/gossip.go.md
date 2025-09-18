
# gossip 概念

## 过程

![[Pasted image 20250916091816.png]]

### pull 和 push
在 Gossip 协议下，网络中两个节点之间有三种通信方式:

- Push: 节点 A 将数据 (key,value,version) 及对应的版本号推送给 B 节点，B 节点更新 A 中比自己新的数据
- Pull：A 仅将数据 key, version 推送给 B，B 将本地比 A 新的数据（Key, value, version）推送给 A，A 更新本地
- Push/Pull：与 Pull 类似，只是多了一步，A 再将本地比 B 新的数据推送给 B，B 则更新本地

注意，gossip 会永远持续，来不断的交换更新的数据。存在只推送增量数据和推送全量数据两种模式。

### view
每个节点只知道少数几个和它相邻的节点，称为view。节点不会把消息给回去。这样，实际在全局的效果是，没有回路消息，但是确实有冗余，即一个节点会从不同的节点身上接收到同一个消息。
在 Gossip 协议中，View 是 **动态变化** 的：

- 新节点加入网络时，会从引导节点那里获取初始 view。
- 节点之间通信时，会交换部分视图内容（比如 “我认识谁”）。
- 周期性地替换/更新 view 中的节点，以保持网络健康和连通性。
- 检测到某个节点失联/长时间无响应，可能从 view 中移除。

### active and passive
这个概念很少被提到，不过从图中可以看出，active 的就是发消息的那个，希望得到数据更新的。

## 在web3中

说是 bitcoin 用了gossip。geth 用的是 [devp2p](https://github.com/ethereum/devp2p) ,还有提到 [# Kademlia](https://blog.csdn.net/han0373/article/details/80494437) 的。总之先把这里的gossip 的应用搞定吧。


# lib p2p

整个文件都是对 pubsub "github.com/libp2p/go-libp2p-pubsub" 的包装。

## 如何使用
```go
package main

import (
    "context"
    "fmt"
    pubsub "github.com/libp2p/go-libp2p-pubsub"
    libp2p "github.com/libp2p/go-libp2p"
)

func main() {
    ctx := context.Background()

    // 创建 libp2p host
    host, err := libp2p.New()
    if err != nil {
        panic(err)
    }

    // 创建 pubsub 实例，使用 Gossipsub 协议
    ps, err := pubsub.NewGossipSub(ctx, host)
    if err != nil {
        panic(err)
    }

    // 加入一个主题
    topic, err := ps.Join("example-topic")
    if err != nil {
        panic(err)
    }

    // 订阅该主题
    sub, err := topic.Subscribe()
    if err != nil {
        panic(err)
    }

    // 启动一个 goroutine 接收消息
    go func() {
        for {
            msg, err := sub.Next(ctx)
            if err != nil {
                fmt.Println("Error reading message:", err)
                return
            }
            fmt.Printf("Received message: %s\n", string(msg.Data))
        }
    }()

    // 发布消息
    topic.Publish(ctx, []byte("Hello from libp2p pubsub!"))
}
```

它使用了发布-订阅模型：
```
Publisher ---> [Topic: "news"] ---> Subscriber A 
                           | +- --> Subscriber B
```
所有订阅topic的人（即subscriber）都会受到这个消息。
publisher 不和 Subscriber 接触，仅仅是往 Topic 上发。

这是一个异步调用过程。可以发现跟 emit(event) 和 on_event 基本一致，不过它的模型范围更广。

上面的例子里，自己的线程既是publisher，做了 topic.Publish，也是subcriber。



