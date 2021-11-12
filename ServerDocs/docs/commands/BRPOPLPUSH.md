## BRPOPLPUSH source destination timeout

    自 2.2.0 起可用。
    时间复杂度：O(1)。

[BRPOPLPUSH](BRPOPLPUSH.md) 是 [RPOPLPUSH](RPOPLPUSH.md) 的阻塞版本。当 source 包含元素的时候，这个命令表现得跟 [RPOPLPUSH](RPOPLPUSH.md) 一模一样。在 [MULTI](MULTI.md)/[EXEC](EXEC.md) 块中使用时，此命令的行为与 [RPOPLPUSH](RPOPLPUSH.md) 完全相同。当 source 为空时，Redis 将会阻塞这个连接，直到另一个客户端 push 元素进入或达到 timeout 时限。timeout 为零时，能用于无限期阻塞客户端。

根据 Redis 6.2.0，[BRPOPLPUSH](BRPOPLPUSH.md) 被认为已弃用。请使用新代码中的 [BLMOVE](BLMOVE.md)。

有关详细信息，请参阅 [RPOPLPUSH](RPOPLPUSH.md) 。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 元素从 source 中弹出，并压入 destination 中。如果达到 timeout 时限，会返回一个 [Null reply](../topics/protocol.md#null-bulk-string) 。

---

### History

- &gt;= 6.0 : timeout 参数改为 double 而不是 integer。

---

### Pattern: Reliable queue

请参阅 [RPOPLPUSH](RPOPLPUSH.md) 文档中的模式描述。

---

### Pattern: Circular list

请参阅 [RPOPLPUSH](RPOPLPUSH.md) 文档中的模式描述。