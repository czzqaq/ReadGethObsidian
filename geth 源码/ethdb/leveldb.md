
在`ethdb/leveldb/leveldb.go` 中。对开源库[leveldb](https://pkg.go.dev/github.com/ethereum/go-ethereum/ethdb/leveldb)的包装


这是一个键值对数据库。key-value database。非SQL 的，更类似于redis，但可以保存到硬盘。


使用方法：
```go
  

import (
    "github.com/syndtr/goleveldb/leveldb"
)

// db *leveldb.DB
db, err := leveldb.OpenFile(file, options)
db.Put(key, value, nil)
db.Get(key, nil)
```

