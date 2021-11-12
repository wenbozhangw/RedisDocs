## WATCH key [key ...]

    自 2.2.0 起可用。
    时间复杂度: 对于每个 key O(1)。

标记要监听的给定 key，以便有条件地执行 [transaction](../topics/transactions.md) 。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) : always OK.