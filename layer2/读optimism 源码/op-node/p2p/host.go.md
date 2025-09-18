
# ExtraHostFeatures

## peerstore 
### 代码中

继承 host.Host，它是libp2p 的一个组件，主要是为了它的 peerstore 相关功能。
例如下面两处：

```go
ps, err := store.NewExtendedPeerstore(context.Background(), log, clock.SystemClock, basePs, conf.Store, scoreRetention)
ps.AddPrivKey(pid, conf.Priv);
ps.AddPubKey(pid, pub)
```
新建了一个自己包装的ps, 然后用密钥初始化。


```go
e.Peerstore().AddAddrs(addr.ID, addr.Addrs, time.Hour*24*7)
```
添加一个连接。


### 用法

在[官方网址](https://pkg.go.dev/github.com/libp2p/go-libp2p-peerstore)上，定义为：
> An object to manage peers, their addresses, and other metadata about them.


用peer store 可以产生一个peer（p2p 连接中的另一端节点），peer 完成一个链接，下面的代码来自 [libp2p-doc](https://docs.libp2p.io/guides/getting-started/go/)

```go
import (
    ...

    "github.com/libp2p/go-libp2p"
    peerstore "github.com/libp2p/go-libp2p/core/peer"
    "github.com/libp2p/go-libp2p/p2p/protocol/ping"
    multiaddr "github.com/multiformats/go-multiaddr"
)

func main() {
    ...
    fmt.Println("libp2p node address:", addrs[0])

    // if a remote peer has been passed on the command line, connect to it
    // and send it 5 ping messages, otherwise wait for a signal to stop
    if len(os.Args) > 1 {
        addr, err := multiaddr.NewMultiaddr(os.Args[1])
        if err != nil {
            panic(err)
        }
        peer, err := peerstore.AddrInfoFromP2pAddr(addr)
        if err != nil {
            panic(err)
        }
        if err := node.Connect(context.Background(), *peer); err != nil {
            panic(err)
        }
        fmt.Println("sending 5 ping messages to", addr)
        ch := pingService.Ping(context.Background(), peer.ID)
        for i := 0; i < 5; i++ {
            res := <-ch
            fmt.Println("got ping response!", "RTT:", res.RTT)
        }
    } else {
        // wait for a SIGINT or SIGTERM signal
        ch := make(chan os.Signal, 1)
        signal.Notify(ch, syscall.SIGINT, syscall.SIGTERM)
        <-ch
        fmt.Println("Received signal, shutting down...")
    }

    // shut the node down
    if err := node.Close(); err != nil {
        panic(err)
    }
}
```



## tcp 配置

基本上这段代码就是在搞 `libp2p.New(opts...)`

```go


	var connGtr gating.BlockingConnectionGater
	connGtr, err = gating.NewBlockingConnectionGater(conf.Store)
	if err != nil {
		return nil, fmt.Errorf("failed to open connection gater: %w", err)
	}
	connGtr = gating.AddBanExpiry(connGtr, ps, log, clock.SystemClock, metrics)
	connGtr = gating.AddMetering(connGtr, metrics)

	connMngr, err := DefaultConnManager(conf)
	if err != nil {
		return nil, fmt.Errorf("failed to open connection manager: %w", err)
	}

	listenAddr, err := addrFromIPAndPort(conf.ListenIP, conf.ListenTCPPort)
	if err != nil {
		return nil, fmt.Errorf("failed to make listen addr: %w", err)
	}
	tcpTransport := libp2p.Transport(
		tcp.NewTCPTransport,
		tcp.WithConnectionTimeout(time.Minute*60)) // break unused connections
	// TODO: technically we can also run the node on websocket and QUIC transports. Maybe in the future?

	opts := []libp2p.Option{
		libp2p.Identity(conf.Priv),
		// Explicitly set the user-agent, so we can differentiate from other Go libp2p users.
		libp2p.UserAgent(conf.UserAgent),
		tcpTransport,
		libp2p.WithDialTimeout(conf.TimeoutDial),
		// No relay services, direct connections between peers only.
		libp2p.DisableRelay(),
		// host will start and listen to network directly after construction from config.
		libp2p.ListenAddrs(listenAddr),
		libp2p.ConnectionGater(connGtr),
		libp2p.ConnectionManager(connMngr),
		//libp2p.ResourceManager(nil), // TODO use resource manager interface to manage resources per peer better.
		libp2p.Peerstore(ps),
		libp2p.BandwidthReporter(reporter), // may be nil if disabled
		libp2p.MultiaddrResolver(madns.DefaultResolver),
		// Ping is a small built-in libp2p protocol that helps us check/debug latency between peers.
		libp2p.Ping(true),
	}
	if conf.NAT {
		// Help peers with their NAT reachability status, but throttle to avoid too much work.
		opts = append(opts,
			libp2p.NATManager(basichost.NewNATManager),
			libp2p.EnableNATService(),
			libp2p.AutoNATServiceRateLimit(10, 5, time.Second*60))
	}
	opts = append(opts, conf.HostMux...)
	if conf.NoTransportSecurity {
		opts = append(opts, libp2p.Security(insecure.ID, insecure.NewWithIdentity))
	} else {
		opts = append(opts, conf.HostSecurity...)
	}
	h, err := libp2p.New(opts...)
```


包括了：
### 传输

- `tcp.NewTCPTransport` + `WithConnectionTimeout`  

### 连接管理
- **ConnManager** (`DefaultConnManager`)
    - 维持连接数在 _low ↔ high watermark_ 之间
    - 超时、打分低的 peer 会被自动裁剪
- **BlockingConnectionGater**
    - 读取 Bad-Peer / Bad-IP 列表（持久化于 Datastore）
    - 决定“是否允许拨号 / 接收入站 / 改协议 …”
    - 再包两层装饰器：
        1. `AddBanExpiry` → 能根据封禁到期时间自动解禁
        2. `AddMetering` → 为 Prometheus 记录每一步决策

### 加密 / 认证

根据配置动态选择

- Noise (`noise.New`)
- TLS 1.3 (`tls.New`)

### peerstore 功能
见 上面的peerstore


### 性能 / 调试

- `libp2p.BandwidthReporter(reporter)` —— 统计字节收发
- `libp2p.Ping(true)` —— 内置 `/ipfs/ping/1.0.0` 协议
    - 额外的 `PingService` 在 goroutine 里持续测 RTT，可上报 Metrics

