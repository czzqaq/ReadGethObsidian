#概念 

# 说明
一种加密哈希算法

BLAKE2是一个比MD5、SHA-1、SHA-2和SHA-3更快的加密散列函数，但至少和最新的标准SHA-3一样安全。BLAKE2以其高速、安全和简单的特点被许多项目采用。


提供checksum 功能：

```go
// Sum512 returns the BLAKE2b-512 checksum of the data.
func Sum512(data []byte) [Size]byte
```

Checksum（校验和）是一种通过对一段数据进行数学计算，生成固定长度的输出值（通常是一个哈希值）的技术。它的主要作用是用于验证数据的完整性。

- Checksum 是从输入数据（如文件、消息或数据块）中计算出来的一个简短、唯一的值。
- 它不会直接透露数据的内容，而是提供了一种验证数据是否被篡改或损坏的简单方法。

在代码中，`Sum512` 函数返回的就是一种特定类型的 checksum，它是由 BLAKE2b 算法计算出来的固定长度（512 位）的值。

也就是输出一个固定长度（比如512bit）的哈希。


另外还提供了一个压缩函数，其实就是类似于XXHash 的 digest。

### **哈希与压缩数据块的关系**

#### **1. 普通哈希**

- 普通哈希（如 MD5 或 SHA-256）通常一次性对整个输入数据进行处理，直接输出一个固定长度的哈希值。
- 如果输入数据很大，普通哈希函数会将数据分块处理，但用户通常看不到这些中间状态，最终只输出一个固定的哈希值。

#### **2. 分块哈希（如 BLAKE2b）**

- 像 BLAKE2b 这样的哈希算法可以分块处理数据，这种设计在处理大数据时非常高效。
- 每个块的数据会被送入压缩函数进行处理，压缩函数会更新中间的状态值（`h`）。
- 这个中间状态（`h`）就像是一个“临时的哈希结果”，不断根据新输入的数据块更新，直到所有数据块都处理完毕，最终输出一个固定长度的哈希值。