# 作用
```go
// journal contains the list of state modifications applied since the last state
// commit. These are tracked to be able to be reverted in the case of an execution
// exception or request for reversal.
type journal struct {
	entries []journalEntry         // Current changes tracked by the journal
	dirties map[common.Address]int // Dirty accounts and the number of changes

	validRevisions []revision
	nextRevisionId int
}
```

## 改变过的address 的管理
它提供了一个对 address 的管理。

```go
// append inserts a new modification entry to the end of the change journal.
func (j *journal) append(entry journalEntry) {
	j.entries = append(j.entries, entry)
	if addr := entry.dirtied(); addr != nil {
		j.dirties[*addr]++
	}
}
```
这个函数的调用：
![[Pasted image 20250327083139.png]]

总之就是关于账户状态：
storage、nonce、code、balance

账户进了accessList

以及账户上彻底的新建等。

## journalEntry
定义了不同的 journalEntry 
```go
// journalEntry is a modification entry in the state change journal that can be
// reverted on demand.
type journalEntry interface {
	// revert undoes the changes introduced by this journal entry.
	revert(*StateDB)

	// dirtied returns the Ethereum address modified by this journal entry.
	dirtied() *common.Address

	// copy returns a deep-copied journal entry.
	copy() journalEntry
}

```

revert 指把入参StateDB 中相应的状态恢复到touch 之前。在journalEntry 中保存之前的状态，这像是一个访问者模式。
绝大多数 dirtied 函数都返回涉及到的地址。不过比如 refundChange、transientStorageChange 等journalEntry的实现，它们的dirtied 返回nil，根据 [[StateDB.go#Finalize]] 中的描述，这些修改不涉及storage object，也就是不涉及world state。 

总结如[[#journalEntry 实现]]

# journalEntry 实现

## **分类 1：账户生命周期变更**

这些变更影响 **账户的创建和销毁**，它们会添加或移除 `stateObject`。

|变更类型|`revert()` 行为|`dirtied()` 返回值|
|---|---|---|
|`createObjectChange`|**删除** 该账户的 `stateObject`|账户地址|
|`selfDestructChange`|**恢复** 该账户的 `selfDestructed` 状态|账户地址|

✅ **特点**：

- `createObjectChange` 和 `selfDestructChange` 互为反向操作，一个用于 **创建** 账户，一个用于 **销毁** 账户。
- 这些变更涉及 `stateObjects`，直接影响 **世界状态（World State）**。

---

## **分类 2：账户标记变更**

这些变更 **不会立即影响账户的存储或余额**，但它们会影响后续交易执行。

|变更类型|`revert()` 行为|`dirtied()` 返回值|
|---|---|---|
|`createContractChange`|取消 `stateObject.newContract` 标记|`nil`|
|`touchChange`|无操作|账户地址|

✅ **特点**：

- `createContractChange` **仅标记** 合约创建，**不会立即影响存储或余额**，但影响 **同一交易内的合约自毁逻辑**。
- `touchChange` 仅用于 **标记账户被访问**，`revert()` **无操作**。

---

## **分类 3：账户状态变更**

这些变更 **修改了账户核心状态**（余额、nonce、存储、代码）。

|变更类型|`revert()` 行为|`dirtied()` 返回值|
|---|---|---|
|`balanceChange`|还原账户的 `balance`|账户地址|
|`nonceChange`|还原账户的 `nonce`|账户地址|
|`storageChange`|还原账户的存储值 `setState()`|账户地址|
|`codeChange`|还原合约代码 `setCode()`|账户地址|

✅ **特点**：

- 这些变更 **直接修改了账户的 World State**（如 `balance`、`nonce`、`storage`、`code`）。
- `storageChange` 和 `codeChange` 影响的是 **合约账户**，但它们本质上与 `balanceChange` 和 `nonceChange` 属于同一类，即 **账户状态变更**。

---

## **分类 4：访问列表变更**

这些变更影响 **EIP-2929 访问列表**，但 **不影响账户状态**。

|变更类型|`revert()` 行为|`dirtied()` 返回值|
|---|---|---|
|`accessListAddAccountChange`|从 `accessList` 中删除该账户|`nil`|
|`accessListAddSlotChange`|从 `accessList` 中删除该存储槽|`nil`|

✅ **特点**：

- 访问列表变更 **仅影响 Gas 计算**，但 **不会影响账户的存储或余额**。
- 由于 **EIP-2929 规定**，如果一个 `slot` 被加入访问列表，那么它的账户一定也已经加入访问列表，因此 `revert()` 可以 **直接移除账户** 而不会影响一致性。

---

## **分类 5：交易日志和退款**

这些变更影响 **交易中的日志和退款信息**，但 **不会影响账户状态**。

|变更类型|`revert()` 行为|`dirtied()` 返回值|
|---|---|---|
|`refundChange`|还原 `refund` 变量的值|`nil`|
|`addLogChange`|从 `logs` 中移除最后一个日志，减少 `logSize`|`nil`|

✅ **特点**：

- 这些变更 **不会影响账户的 World State**，但它们对 **交易回滚** 仍然至关重要。
- `addLogChange` 直接影响 `logs` 数据结构，但不会影响 `stateObject`。

