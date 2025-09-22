# 作用说明

基于LibP2P stream，实现了同步功能。不同node 之间可以传输 `L2 ExecutionPayload`

这份代码让 Optimism 节点能够：
- 作为**服务器**：受控速率地向其他节点按区块号提供执行层数据；
- 作为**客户端**：并行、背压、按区间地从多个 peer 拉取缺失区块，并通过父哈希验证保证链一致性。


## 作为client，请求
1. 我（节点 A）发现自己缺块
	1. 输入 `[start, end]`，形成一个一个 **区块号**，放进本地的“待拉取队列”。
2. 提供 AddPeer 接口。这个接口在初始化时声明sync 的peer 有哪些。
3. 对每个 peer 起一个 goroutine/线程专门和它打交道，带上一些节流（限速）规则
	1. 在 LibP2P 之上 **打开一条流**（stream）。
	2. 写入 `uint64` 区块号 `N`。
	3. 关写半边，等对方回数据。

## 作为 server
1. 先看看请求合不合理（别让人要创世块以前、未来块以后）。
2. 本地数据库里查有没有区块 `N`。
	1. 有：构造响应
3. 关闭流

## 作为 client，收到请求

1. 解码 + 基本合法性检查
2. **父哈希信任链检查**：
	1. 如果我已经确信这块的 `hash` 或它的 `parentHash` 在主链上 → 立即“提拔”为正式区块。
	2. 否则先丢进 **quarantine**（隔离区缓存），等待以后拿到父块再一起验证。
3. **打分 / 节流**：
	1. 有效响应 → 给节点 B 加分。
	2. 错误/超时/垃圾数据 → 减分、并且在限速器里“扣更多 Token”，让它稍冷静。
4. 循环，请求更低高度的block。直到队列空

# 功能模块
## LibP2P stream

`github.com/libp2p/go-libp2p/core/network/network`
模块的 network.Stream 

1. host.NewStream(ctx, peerID, protoID…) 
	主动建立一条双向流
2. SetStreamHandler(protoID, func(Stream))
	说明这个流的处理细节
3. binary.write(stream, ...) 
	写数据的具体方法

## 速率限制

使用的是 token-bucket 。
具体为 "golang.org/x/time/rate" 这个库

Limiter 是天生支持多线程访问的。
```go
package main

import (
	"context"
	"fmt"
	"time"

	"golang.org/x/time/rate"
)

func main() {
	// 每秒 5 个请求，允许瞬时突发 10 个
	lim := rate.NewLimiter(5, 10)

	do := func(i int) {
		ctx, cancel := context.WithTimeout(context.Background(), time.Second)
		defer cancel()

		if err := lim.Wait(ctx); err != nil { // 拿 1 个 token
			fmt.Println("timeout:", i, err)
			return
		}
		fmt.Println("do job", i, "at", time.Now())
	}

	for i := 0; i < 20; i++ {
		go do(i)
	}

	time.Sleep(4 * time.Second)
}
```

涉及到函数：


1. func NewLimiter(r Limit, b int) *Limiter
	1. 创建一个新的rate limiter
2. Allow()
	1. 马上尝试拿 1 个令牌；拿到返回 `true`，否则 `false`
3. Reserve（）
	1. 预订令牌，返回 `Reservation` 描述需要等待多久。
4. Wait（ctx）
	1. 阻塞版的allow ，阻塞直到令牌可用（相当于 allow() 返回true）或者ctx 过期。


### peer 打分
```go
type SyncPeerScorer interface {
    onValidResponse(id peer.ID)
    onResponseError(id peer.ID)
    onRejectedPayload(id peer.ID)
}
type ScoreBook interface {
    GetPeerScores(id peer.ID) (store.PeerScores, error)
    SetScore(id peer.ID, diff store.ScoreDiff) (store.PeerScores, error)
}
```

主要是上面接口实现的，具体加减的分数套用了 `./p2p/store/iface.go` 中 的规范，store 的意思是储存，peer 分数要持久化的。anyway，不太重要了。



