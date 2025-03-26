整个world state 。原理见：[[4.1 world state]]

是stateObject 的集合。类似于：
$$
stateObject = \sigma(a) = statedb[a]
$$
（不是等于的关系，毕竟程序里有很多其他内容）


# 代码
## getStateObject
```go
// getStateObject retrieves a state object given by the address, returning nil if
// the object is not found or was deleted in this execution context.
func (s *StateDB) getStateObject(addr common.Address) *stateObject {
	// Prefer live objects if any is available
	if obj := s.stateObjects[addr]; obj != nil {
		return obj
	}
	// Short circuit if the account is already destructed in this block.
	if _, ok := s.stateObjectsDestruct[addr]; ok {
		return nil
	}
	s.AccountLoaded++

	start := time.Now()
	acct, err := s.reader.Account(addr)
	if err != nil {
		s.setError(fmt.Errorf("getStateObject (%x) error: %w", addr.Bytes(), err))
		return nil
	}
	s.AccountReads += time.Since(start)

	// Short circuit if the account is not found
	if acct == nil {
		return nil
	}
	// Schedule the resolved account for prefetching if it's enabled.
	if s.prefetcher != nil {
		if err = s.prefetcher.prefetch(common.Hash{}, s.originalRoot, common.Address{}, []common.Address{addr}, nil, true); err != nil {
			log.Error("Failed to prefetch account", "addr", addr, "err", err)
		}
	}
	// Insert into the live set
	obj := newObject(s, addr, acct)
	s.setStateObject(obj)
	s.AccountLoaded++
	return obj
}
```

## getState
详见：[[state_object.go#getState]]
