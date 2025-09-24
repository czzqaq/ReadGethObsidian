# 说明

基于 pebble db 。
在硬盘中记录 **L1 block ID →（L1 block hash，corresponding safe L2 hash, corresponding safe L2 number）** 。

其记录函数：
```go
func (d *SafeDB) SafeHeadUpdated(safeHead eth.L2BlockRef, l1Head eth.BlockID) error {
    d.m.Lock()
    defer d.m.Unlock()

    d.log.Info("Record safe head", "l2", safeHead.ID(), "l1", l1Head)

    batch := d.db.NewBatch()
    defer batch.Close()

    if err := batch.Set(safeByL1BlockNumKey.Of(l1Head.Number), safeByL1BlockNumValue(l1Head, safeHead.ID()), d.writeOpts); err != nil {
        return fmt.Errorf("failed to record safe head update: %w", err)
    }


    batch.Commit(d.writeOpts)
    return nil
}
```

safeByL1BlockNumKey.Of 和 safeByL1BlockNumValue 都是一些特殊的序列化规则。

# pebble db
go 的，kv 数据库

## 使用例
```go
package main

import (
	"github.com/cockroachdb/pebble"
	"log"
)

func main() {
	db, err := pebble.Open("mydb", &pebble.Options{})
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()

	// 写入键值
	if err := db.Set([]byte("hello"), []byte("world"), nil); err != nil {
		log.Fatal(err)
	}

	// 读取键值
	value, closer, err := db.Get([]byte("hello"))
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("Value: %s", value)  // expect: world
	closer.Close()
}
```

