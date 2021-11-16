## UNWATCH

    起始版本：2.2.0.
    时间复杂度：O(1).

刷新一个 [transaction](../topics/transactions.md) 中已被监听的所有 key。

如果执行 [EXEC](exec.md) 或 [DISCARD](discard.md)，则不需要手动执行 [UNWATCH](unwatch.md) 。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) : always OK.