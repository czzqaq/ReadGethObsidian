#概念 

# 简介
## 是什么

RPC（Remote Procedure Call）是一种允许程序调用远程服务器上的函数，像调用本地函数一样工作的机制。它不算是个协议，是技术概念名词。

- **客户端 (Client)**：用于发送请求给远程服务端。函数的调用者。
- **服务端 (Server)**：处理来自客户端的请求并返回结果。函数的执行者。

RPC 要让远程调用时，调用者（client）本身不需要管理HTTP 的协议等，甚至感受不到远程调用的逻辑。

## 有什么用

在 `geth` 中，RPC 是以太坊节点和外部程序（如 DApp 或其他服务）交互的主要机制，支持多种协议（如 HTTP、WebSocket 和 IPC）。


以太坊是一个点对点网络，每个节点都可以通过 RPC 接口提供以下服务：
- 查询链上数据（如区块、交易、账户余额）。
- 提交交易。
- 订阅链上事件（如新块、交易日志）。
- 管理节点状态（如启动或停止挖矿）。



# 过程

1. 远程服务之间建立通讯协议

2. 寻址：服务器（如主机或IP地址）以及特定的端口，方法的名称名称是什么

3. 通过序列化和反序列化进行数据传递


## 通信建立

套在HTTP 等协议下面。协议支持：
- HTTP：基于标准 HTTP 协议的 RPC 通信。
- WebSocket：支持双向通信，常用于事件推送。
- IPC：基于本地文件套接字（Unix Domain Sockets 或 Windows Named Pipes）的通信，速度最快但只能用于本地。


## 序列化
使用：[JSON-RPC协议](https://wiki.geekdream.com/Specification/json-rpc_2.0.html)
例如，id 是客户端的标识，每个客户端唯一，没有id 字段表示这是一个通知。params 是调用方法时的argument。
例如：
client --> 
```json
{"jsonrpc": "2.0", "method": "subtract", "params": {"subtrahend": 23, "minuend": 42}, "id": 3}

```
server <---
```json
{"jsonrpc": "2.0", "result": 19, "id": 3}
```

也可以没有返回值：
```json
 {"jsonrpc": "2.0", "method": "update", "params": [1,2,3,4,5]}
```
