# 功能
负责Rollup 节点进程的的启动、停止和运行。各种功能的协调运行。

## 涉及组件
- 事件系统 [[system.go]]
- tracer 事件记录
- metric 记录
- L1 订阅。[[l1_client.go]]
- L2 的 geth 引擎。 [[engine_client.go]]
- L1 配置加载 ，包括 L1 safe head 的确定。以及Protocol version(见[[layer2/读optimism 源码/op-node/node/superchain.go]])
- p2p 相关，见 [[layer2/读optimism 源码/op-node/p2p/node.go]]
- conductor, 见[[conductor.go]]
- AltDA ,见[[op-alt-da]]
- safe DB，见 [[safedb]]
- RPC ，实现对 rollup 的RPC 控制（相对应的，还有cli 版本），见[[server.go]] 

