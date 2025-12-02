
# 内容
就是启动batcher 服务。主要是开启3个channel 和 4个 loop。

## 四个loop

### blockLoadingLoop

```
每 PollInterval:
    syncStatus = getSyncStatus() // sequencer 的status
    blocksToLoad = syncAndPrune(syncStatus) 

    若 blocksToLoad != nil:
        loadBlocksIntoState(start,end)
        sendToThrottlingLoop(pendingBytes)
    trySignal(publishSignal)
```

其中:

`syncAndPrune` “prunes blocks and channels from its internal state which are no longer required”。 详见：[[sync_actions.go]] 获得具体哪些需要被prune，而从channel 中删除的意思见 [[channel_manager.go]]

`loadBlocksIntoState`: 它的block 来源是L2 client(客户端)，不在本optimism 的项目中实现。block 写入到[[channel_manager.go]] 


### publishingLoop

```
for range publishSignal:
    if !checkTxpool() { continue }     // 取消逻辑
    publishStateToL1()                 // 循环直到没有更多 txData
```

发送 state 到L1，并等待完成。

其中，
publishStateToL1。发送通过通道：`txmgr`，这是一个op-service，包括了等待回执之类的，但总之核心是 ETHBackend 的接口 `SendTransaction`。这个接口是由 L2 client 提供的。发送数据的来源是 [[channel_manager.go]] （TxData），

等待包括两部分，其核心是 [[#errgroup]] 。等待协程运行全部结束，即收到了receipt,才开始下一个loop。

### receiptsLoop
```

等待接受到回执。
对每个 `TxReceipt[txRef]`：
- 将记账工作委托给 `recordConfirmedTx` / `recordFailedTx`。
```

recordConfirmedTx 的实现见 [[channel_manager.go]]（TxConfirmed）


### throttlingLoop
```

等待 blockLoadingLoop 的信号

for pb := range pendingBytesUpdated:
    throttling = pb > ThrottleThreshold
    对每个 endpoint:
        做 throttle——控制 sequencer 产生的L2 data 数量
```


其中，throttle 是通过给endpoint 发送 rpc 的方式，调用`setMaxDASize` 的方法。编程上很容易理解。

为什么要做这个 throttling ：
#### Data Availability Backlog(DA 积压)


> A chain can potentially experience an influx of large transactions whose data availability requirements exceed the total throughput of the data availability layer. While this situation might resolve on its own in the long term through the data availability pricing mechanism, in practice this feedback loop is too slow to prevent a very large backlog of data from being produced, even at a relatively low cost to whomever is submitting the large transactions. In such circumstances, the safe head can fall significantly behind the unsafe head, and the time between seeing a transaction (and charging it a given L1 data fee) and actually posting the transaction to the data availability layer grows larger and larger. Because DA costs can rise quickly during such an event, the batcher can end up paying far more to post the transaction to the DA layer than what can be recovered from the transaction's data fee. To prevent a significant DA backlog, the batcher can instruct the block builder (via op-geth's miner RPC API) to impose thresholds on the total DA requirements of a single block, and/or the maximum DA requirement of any single transaction. In the happy case, the batcher instructs the block builder to impose a block-level DA limit of OP_BATCHER_THROTTLE_ALWAYS_BLOCK_SIZE, and imposes no additional limit on the DA requirements of a single transaction. But in the case of a DA backlog (as defined by OP_BATCHER_THROTTLE_THRESHOLD), the batcher instructs the block builder to instead impose a (tighter) block level limit of OP_BATCHER_THROTTLE_BLOCK_SIZE, and a single transaction limit of OP_BATCHER_THROTTLE_TRANSACTION_SIZE.

如果 L2 上突然出现大量“大体积”交易，而它们需要写入的数据量已经超过了 DA 层（例如 L1 calldata／blob）的吞吐能力，就会出现“数据可用性积压”。  
理论上 DA 费用会随着需求上升而自动变贵，从而抑制大体积交易，但价格调节有延迟，来不及阻止短时间内的海量数据涌入。

积压带来的两个主要问题：

1. **安全头（safe head）严重落后**  
    Sequencer 生成的新块（unsafe head）越来越多，但它们迟迟发不上 L1，导致链的“已确认”部分拖后很远。
    
2. **批次发布者（batcher）倒贴钱**  
    当初向用户收取的 L1 数据费基于“当时”的 DA 价格计算。如果真正发布时 DA 价格暴涨，batcher 可能要支付远高于它收的钱。


## channel

不同的loop 之间用channel 通信，例如，当 blockLoadingLoop 检测到 sequencer 的状态合适后，读取了block，发送给 Publish Loop 信号，来说明：“来活儿了”。它比较类似于 emit 和 on event 的驱动，平时，这些loop 的go routine 就单纯loop 着。

### 3个 channel

```go
    go l.receiptsLoop(l.wg, receiptsCh)                                           // ranges over receiptsCh channel

    go l.publishingLoop(l.killCtx, l.wg, receiptsCh, publishSignal)               // ranges over publishSignal, spawns routines which send on receiptsCh. Closes receiptsCh when done.

    go l.blockLoadingLoop(l.shutdownCtx, l.wg, unsafeBytesUpdated, publishSignal) // sends on unsafeBytesUpdated (if throttling enabled), and publishSignal. Closes them both when done
    
   go l.throttlingLoop(l.wg, unsafeBytesUpdated) // ranges over unsafeBytesUpdated channel
```

| channnel           | in               | out            |
| ------------------ | ---------------- | -------------- |
| unsafeBytesUpdated | blockLoadingLoop | throttlingLoop |
| publishSignal      | blockLoadingLoop | publishingLoop |
| receiptsCh         | publishingLoop   | receiptsLoop   |


# 语法
## waitgroup
`sync.WaitGroup` 是 Go 提供的一个用于 **等待一组 goroutine 完成执行的同步**。它常用于多个并发任务之间的协作，确保主线程在所有子任务完成之后再继续执行。

### 例子
```go
var wg sync.WaitGroup

wg.Add(1) // 增加一个计数器
go func() {
    defer wg.Done() // 子 goroutine 完成时递减计数器
    fmt.Println("Hello from goroutine")
}()

wg.Wait() // 等待所有计数器归零
fmt.Println("All goroutines finished")
```
输出：
```
Hello from goroutine
All goroutines finished
```

应该很好理解，增加一个计数器，每wg.Done() 一次，都计数减1，当计数为0时，wg.Wait() 就不阻塞了。

### 注意

wg 的计数为负时，panic。
线程安全。
wg.Add 和Wait() 之间存在竞争关系。该在协程前调用 wg.Add。例如

这是合法的：

```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
	wg.Add(1) // ✅ 在 goroutine 启动前调用
	go func() {
		defer wg.Done()
		// do something
	}()
}

wg.Wait()
```

这不合法：
```go
var wg sync.WaitGroup

for i := 0; i < 10; i++ {
	go func() {
		wg.Add(1) // ❌ 不安全：Add 与 Wait 并发执行
		defer wg.Done()
	}()
}

wg.Wait()
```

wait 和 add 必须在一个线程里，不然子携程还没运行，主线程已经wait 调用，发现了值是0，就直接放开了。

## errgroup

它在 `sync.WaitGroup` 的基础上增加了 **错误处理** 和 **Context 取消支持**。

`errgroup.Group` 通常用于如下场景：

- 并行启动多个 goroutine 。
- 如果任何一个goroutine返回错误或 panic，则取消所有其他 goroutine 的执行。
- 等待所有 goroutine 完成，统一处理错误或 panic。

### 例子

下面是一个使用例。它用于场景 2，即对其他routine 的感知。
```go
func main() {
	ctx := context.Background()
	group, ctx := errgroup.WithContext(ctx)
	group.SetLimit(5) // 最多5个同时运行

	for i := 0; i < 100; i++ {
		i := i // 防止闭包引用问题
		group.Go(func() error {
			select {
			case <-ctx.Done():
				return ctx.Err() // 提前退出
			default:
				// 模拟任务
				if i == 42 {
					return fmt.Errorf("error at task %d", i)
				}
				fmt.Println("Task", i, "done")
				return nil
			}
		})
	}

	if err := group.Wait(); err != nil {
		fmt.Println("Stopped early due to:", err)
	}
}
```
这里实现了context 和 协程的共同控制，任何一端都能把全部 routine 停止，当然也可以运行完了停止。


### context

context 本质就是一个读写安全的信号。并提供了超时、cancel等控制方式。

例如，这里是一个cancel 的例子。任何触发了ctx.cancel() 的过程，都往 channel 上给了信息。
```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    select {
    case <-ctx.Done():
        fmt.Println("Goroutine canceled:", ctx.Err())
    }
}()

// 模拟某些条件触发取消
time.Sleep(time.Second)
cancel()
```

这里是一个通过 context+error group，处理协程协作的场景。
```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
    return errors.New("task 1 failed")
})

g.Go(func() error {
    for {
        select {
        case <-ctx.Done():
            fmt.Println("Task 2 cancelled:", ctx.Err())
            return nil
        default:
            fmt.Println("Task 2 working...")
            time.Sleep(time.Second)
        }
    }
})

_ = g.Wait()
```
### 在代码中
daGroup 和 txQueue 本身都是 err group
#### da group
daGroup 是简单的带 setlimit 的group，用来控制并行的DA 提交任务的数量。
```go
daGroup := &errgroup.Group{}
daGroup.SetLimit(int(l.Config.MaxConcurrentDARequests))

...

goroutineSpawned := daGroup.TryGo(func() error {
	comm, err := l.AltDA.SetInput(l.shutdownCtx, txdata.CallData())
	if err != nil {
		// 错误处理逻辑
		return nil // 注意这里返回 nil 是为了让 errgroup 不认为是致命错误
	}

	// 成功后的后续处理
	l.sendTx(...)
	return nil
})
```

#### txQueue 

启动了一些 等待回复Receipt 的协程，关键是有个context。

```go
// groupContext returns a Group and a Context to use when sending a tx.
//
// If any of the pending transactions returned an error, the queue's shared error Group is
// canceled. This method will wait on that Group for all pending transactions to return,
// and create a new Group with the queue's global context as its parent.
func (q *Queue[T]) groupContext() (*errgroup.Group, context.Context) {
   if q.group != nil {
      _ = q.group.Wait()
   }
   q.group, q.groupCtx = errgroup.WithContext(q.ctx)
  q.group.SetLimit(int(q.maxPending))

    return q.group, q.groupCtx
}
```