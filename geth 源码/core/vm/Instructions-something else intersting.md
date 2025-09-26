# SStore
```go
func opSstore(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	if interpreter.readOnly {
		return nil, ErrWriteProtection
	}
	loc := scope.Stack.pop()
	val := scope.Stack.pop()
	interpreter.evm.StateDB.SetState(scope.Contract.Address(), loc.Bytes32(), val.Bytes32())
	return nil, nil
}
```
用来修改storage，storage 的概念见 [[state_object.go#storage]]

# ADD
```go
func opAdd(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	x, y := scope.Stack.pop(), scope.Stack.peek()
	y.Add(&x, y)
	return nil, nil
}
```
调用的时候，注意传参都是通过栈，返回值也是通过栈。pop() 函数返回的是具体的值，而peek() 则是返回了一个指针，不修改栈本身。`int *.add` 操作的入参是指针。
了解上述语法后，我明白：此函数可以改写为：

```go
func opAdd(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	stack := scope.Stack
	b := stack.pop()          // 栈顶元素
	a := stack.pop()          // 次顶元素

	result := new(big.Int).Add(a, b) // result = a + b

	stack.push(result)       // 把结果压回栈顶

	return nil, nil
}
```

# Keccak256

```go
func opKeccak256(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
	offset, size := scope.Stack.pop(), scope.Stack.peek()
	data := scope.Memory.GetPtr(offset.Uint64(), size.Uint64())

	if interpreter.hasher == nil {
		interpreter.hasher = crypto.NewKeccakState()
	} else {
		interpreter.hasher.Reset()
	}
	interpreter.hasher.Write(data)
	interpreter.hasher.Read(interpreter.hasherBuf[:])

	size.SetBytes(interpreter.hasherBuf[:])
	return nil, nil
}

```
注意，传入参数传入的实际上是memory 上的一段位置，它适用于例如CALL 等需要传入更大的连续的[] 的时候。

