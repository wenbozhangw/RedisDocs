## CLIENT GETNAME

    起始版本：2.6.9。
    时间复杂度：O(1)。

[CLIENT GETNAME](CLIENT GETNAME.md) 返回由 [CLIENT SETNAME](CLIENT SETNAME.md) 设置的当前连接的名称。由于每个新连接开始时都没有关联的名称，如果没有分配名称，则返回 null bulk reply。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 连接名称，如果未设置名称，则为 null bulk reply 。