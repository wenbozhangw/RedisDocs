## EXEC

    自 1.2.0 起可用。

执行 [transaction](../topics/transactions.md) 中所有先前排队的命令并将连接状态恢复为正常。

当使用 [WATCH](WATCH.md)， [EXEC](EXEC.md) 仅在监听的 key 未被修改时才会执行命令，从而允许 [check-and-set mechanism](../topics/transactions.md#optimistic-locking-using-check-and-set) 。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) : 每个元素都是对 atomic transaction 中每个命令的回复。

使用 [WATCH](WATCH.md)，[EXEC](EXEC.md) 时，如果执行被中止， 可以返回 [Null reply](../topics/protocol.md#null-bulk-string) 回复。