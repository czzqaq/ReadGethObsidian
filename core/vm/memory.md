某些操作涉及到了memory 字段。
主要是源码的 `core/vm/memory.go` 相关的内容。
# 概述
```go
// Memory implements a simple memory model for the ethereum virtual machine.
type Memory struct {
	store       []byte
	lastGasCost uint64
}

```
可以发现，memory的实现挺简单，就是一个buffer。关于 lastGasCost,见：[[#memoryGasCost]]

## memory接口

| 方法名           | 功能说明                   |     |
| ------------- | ---------------------- | --- |
| `NewMemory()` | 获取新的内存实例（来自 sync.Pool） |     |
| `Free()`      | 释放内存，归还到内存池            |     |
| `Resize()`    | 动态扩展内存                 |     |
| `Set()`       | 写入任意长度的字节              |     |
| `Set32()`     | 写入 32 字节的 uint256 值    |     |
| `GetCopy()`   | 获取一个内存副本               |     |
| `GetPtr()`    | 获取内存区域的引用（非副本）         |     |
| `Copy()`      | 内存内部数据复制               |     |
| `Len()`       | 当前内存长度                 |     |
| `Data()`      | 获取底层字节数组               |     |
## memory pool
使用了go 内部的pool 概念，见编程
```go
var memoryPool = sync.Pool{
	New: func() any {
		return &Memory{}
	},
}
```
# 程序
## memoryGasCost



# 编程
## sync.Pool

`sync.Pool` 是一个线程安全的池，专门用于存放 **不再使用但可能被复用的对象**。它的主要作用是：

- 减少频繁创建和销毁对象带来的性能开销。
- 缓解垃圾回收器（GC）的负担，提升程序性能，尤其是在高并发场景下。

### 默认的内存管理
如果不用 `Pool`，Go 语言默认采用 **自动内存管理机制**，即使用了内建的 **垃圾回收系统（Garbage Collector, GC）** 来管理内存的分配和释放。

例如，下面的代码中，每次进入for 循环，都会产生一次malloc：
```go
func main() {
	for i := 0; i < 1000000; i++ {
		data := make([]byte, 1024) // 每次都创建新的内存块
		_ = data
	}
}
```

### 使用内存池
上述代码可以改为：
```go
var bufPool = sync.Pool{
	New: func() any {
		return make([]byte, 1024)
	},
}

func main() {
	for i := 0; i < 1000000; i++ {
		data := bufPool.Get().([]byte)
		// 使用 data 做点什么
		bufPool.Put(data) // 使用后归还
	}
}
```

在geth 中：
```go
var memoryPool = sync.Pool{
	New: func() any {
		return &Memory{}
	},
}

func NewMemory() *Memory {
	return memoryPool.Get().(*Memory)
}

func (m *Memory) Free() {
	// 控制归还的大小，避免过大内存泄漏,过大的直接回收。注意这里有点绕，因为默认free，所以有处理的才不被free
	if cap(m.store) <= 16<<10 {
		m.store = m.store[:0]
		m.lastGasCost = 0
		memoryPool.Put(m)
	}
}
```

### slice 的len 和 cap
这里说明代码：
```go
func (m *Memory) Free() {
	// 控制归还的大小，避免过大内存泄漏,过大的直接回收。注意这里有点绕，因为默认free，所以有处理的才不被free
	if cap(m.store) <= 16<<10 {
		m.store = m.store[:0]
		m.lastGasCost = 0
		memoryPool.Put(m)
	}
}
```
 的作用。`m.store = m.store[:0]` 做了什么。
#### 是什么
```
[底层数组] = [元素0][元素1][元素2]...[元素N]
              ↑                      ↑
            len                     cap
```

- `len(slice)`：表示你当前**可以访问的元素数量**。
- `cap(slice)`：表示**最大可以扩展到的元素数量（不分配新内存）**。


```go
s := make([]byte, 2, 5) // 创建长度为2，容量为5的 slice 
fmt.Println(len(s)) // 2 
fmt.Println(cap(s)) // 5
```

```
s = s[:0]
```

这句话的意思是：

- 把 `len(s)` 设为 0，表示“我暂时不用任何元素”；
- 但 **底层数组还在**，`cap(s)` 依然是 5；
- 所以你可以重新向它追加元素：

#### 为什么
假设我们强制 **每次 slice 的长度都等于容量**，那么就会出现比如：每次 `append` 都必须重新分配内存的情况

```go
s := make([]int, 3) // len=3, cap=3
s = append(s, 4)    // 超出 cap，必须重新分配一个更大的底层数组
```

