## RANDOMKEY 

    起始版本：1.0.0。
    时间复杂度：O(1)。

从当前选择的数据库随机返回一个 key。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 随机 key，如果数据库是空的，返回 nil。