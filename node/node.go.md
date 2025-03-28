
看起来基本上是提供网络服务的，

``` go
type Node struct {

    eventmux      *event.TypeMux

    config        *Config

    accman        *accounts.Manager

    log           log.Logger

    keyDir        string        // key store directory

    keyDirTemp    bool          // If true, key directory will be removed by Stop

    dirLock       *flock.Flock  // prevents concurrent use of instance directory

    stop          chan struct{} // Channel to wait for termination notifications

    server        *p2p.Server   // Currently running P2P networking layer

    startStopLock sync.Mutex    // Start/Stop are protected by an additional lock

    state         int           // Tracks state of node lifecycle

  

    lock          sync.Mutex

    lifecycles    []Lifecycle // All registered backends, services, and auxiliary services that have a lifecycle

    rpcAPIs       []rpc.API   // List of APIs currently provided by the node

    http          *httpServer //

    ws            *httpServer //

    httpAuth      *httpServer //

    wsAuth        *httpServer //

    ipc           *ipcServer  // Stores information about the ipc http server

    inprocHandler *rpc.Server // In-process RPC request handler to process the API requests

  

    databases map[*closeTrackingDB]struct{} // All open databases

}
```

关注其中的：

``` go
server        *p2p.Server
```
这是中心。其他的key之类的就是辅助网络运行。

