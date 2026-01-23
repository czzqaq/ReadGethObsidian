# 背景

## reference
- [官方disv5规范](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md)
- [[discovery.go]] 描述了dicv5 的概念。
- [我正在读的repo](https://github.com/sigp/discv5)


# 代码知识
## tokio

在example 里，代码有：
```rust
// build the tokio executor
let mut runtime = tokio::runtime::Builder::new_multi_thread()
    .thread_name("Discv5-example")
    .enable_all()
    .build()
    .unwrap();

// start the discv5 server
runtime.block_on(discv5.start());

// run a find_node query
runtime.block_on(async {
   let found_nodes = discv5.find_node(NodeId::random()).await.unwrap();
   println!("Found nodes: {:?}", found_nodes);
});
```

这里说说所谓 runtime 是什么。

Tokio runtime 是 **异步任务运行环境**，负责调度和执行async/Future。

`discv5.start()` 是一个async 函数，返回的是一个 `impl Future<Output = _>`。

### async
调用时**不**立即运行代码体。它立即返回一个 `Future`。只有当这个 `Future` 被执行器（Executor）轮询（poll）时，代码体才会开始执行。
它通常和 await 配合使用，下面是一个串行执行的例子。
```rust
async fn example(x: u8) -> u8 {
    let y = read_from_io().await;
    let z = calculate(y).await;
    x + z
}
```
通过 tokio 提供Runtime，下面是一个并行的例子：
```rust
use tokio::time::{sleep, Duration};
use std::time::Instant;

// 模拟一个耗时的网络请求
async fn fetch_data(server_name: &str, seconds: u64) -> String {
    sleep(Duration::from_secs(seconds)).await; // 模拟网络延迟，这里会释放 CPU
    println!("成功从 [{}] 获取数据!", server_name);
}

#[tokio::main]
async fn main() {
    println!("--- 开始并发执行 (Concurrent) ---");
    let start = Instant::now();

    // 关键点：这里只是创建了 Future，还没有 await，所以它们还没开始跑
    let task_a = fetch_data("Server A", 2);
    let task_b = fetch_data("Server B", 3);

    // 使用 tokio::join! 宏
    // 它会让 task_a 和 task_b "同时" 被轮询 (poll)
    // 只要其中一个在等待 (sleep)，执行器就会去推另一个
    let (data_a, data_b) = tokio::join!(task_a, task_b);

    let duration = start.elapsed();
    println!("并发执行总耗时: {:?} (预期: max(2,3)=3秒)", duration);
}
```

### 运行时
“运行时”指的是**为了让你的程序跑起来，语言自带的那一套庞大的支撑系统**。
Rust 语言本身只定义了“什么是异步任务（Future）”，但它不负责“怎么执行这些任务”。也就是说，必须依靠第三方环境，才能真的把异步用起来。Tokio 就是这样一个给异步任务的运行时。


### block_on
```rust
tokio::runtime::Builder::new_multi_thread() .thread_name("Discv5-example") .enable_all() // 启用 I/O 和计时器驱动
.build() .unwrap(); // 这样就提供了一个 runtime
```
```rust
runtime.block_on(discv5.start());
```
`block_on` 会阻塞当前线程，直到 `discv5.start()` 这个 Future 执行完成。
### channel 和 Actor 模式

```rust
let channel = mpsc::Sender<ServiceRequest>();
let (callback_send, callback_recv) = oneshot::channel();
channel .send(event)
callback_recv .await
```
oneshot 一种 channel，和 go 的差不多，特点是 one-shot，即sender 只能发一条消息， receiver 只能收一条消息。

mpsc 也是一类channel，更类似订阅模式，multiple producer Single Consumer 。

Actor 模式，比如 RPC 就是其一种应用，不同模块之间互相独立，基于消息进行通信。
下面是我让AI  给我写的一个 demo。
```rust
use tokio::sync::{mpsc, oneshot};

//这是我们要发送给后台的“任务单”
struct Task {
    // 1. 我们要处理的数据
    data: String,
    // 2. 关键点：这是用来接收结果的“回执单” (oneshot发送端)
    //    后台算出结果后，会通过这个通道把 usize 类型的长度发回来
    respond_to: oneshot::Sender<usize>,
}

#[tokio::main]
async fn main() {
    // --- 步骤 1: 建立通往后台服务的“主通道” (mpsc) ---
    // tx (Sender): 用来发任务
    // rx (Receiver): 后台拿着它收任务
    let (tx, mut rx) = mpsc::channel::<Task>(32);

    // --- 步骤 2: 启动后台服务 (模拟 Service Loop) ---
    tokio::spawn(async move {
        println!("[后台] 服务已启动，等待任务...");
        
        // 只要主通道里有任务，就一直循环处理
        while let Some(task) = rx.recv().await {
            println!("[后台] 收到字符串: '{}'", task.data);
            
            // 模拟耗时操作
            let len = task.data.len(); 

            // 关键点：拿出任务单里的“回执单”，把结果发回去
            // send 可能会失败（如果主线程已经不等了），这里简单忽略错误
            let _ = task.respond_to.send(len);
            println!("[后台] 结果已发送");
        }
    });

    // --- 步骤 3: 主线程发起请求 (模拟 find_node) ---
    
    // A. 创建一次性的“回执通道” (oneshot)
    // resp_tx: 给后台，让它发结果
    // resp_rx: 留给自己，用来等结果
    let (resp_tx, resp_rx) = oneshot::channel();

    // B. 填写任务单
    let task = Task {
        data: String::from("Hello Rust"),
        respond_to: resp_tx, // 把回执发送端塞进去！
    };

    // C. 把任务单通过“主通道”发给后台
    println!("[主线程] 发送任务...");
    tx.send(task).await.unwrap();

    // D. 在“回执接收端”死等结果
    println!("[主线程] 等待结果...");
    let result = resp_rx.await.unwrap();

    println!("[主线程] 收到结果: 长度是 {}", result);
}
```

# 代码
## 架构
有一个中心的 service.rs，来处理全部消息。这个service 内监听一个 consumer，并把 producer 给handle 保存。任何一次调用（比如FindNode) 都会获得一个新的 producer（mpsc支持clone producer) ，然后给service 发消息。service 负责调用具体的某个功能函数。


## k bucket
### 普通的k bucket
这里不是一个总结，仅仅帮我回忆一下，为理解discv5 的版本打好基础。
先定义桶。它提供add 和split。一个桶可以储存固定 id 范围的node，如果split，范围减小而提供了不满的桶。桶的集合是路由表，路由表在初始化时，提供一个local id，并建立一个range 为$[0,2^{key_len})$ 的桶包括它。添加节点时，split 包含local id 的桶，丢弃
