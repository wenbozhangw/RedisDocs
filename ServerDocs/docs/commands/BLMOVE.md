## BLMOVE source destination LEFT|RIGHT LEFT|RIGHT timeout

    起始版本：6.2.0。
    时间复杂度：O(1)。

[BLMOVE](BLMOVE.md) 是 [LMOVE](LMOVE.md) 的阻塞版本。当 source 包含元素时，此命令的行为和 [LMOVE](LMOVE.md) 完全相同。在 [MULTI](MULTI.md)/[EXEC](EXEC.md) 块中使用时，此命令的行为与 [LMOVE](LMOVE.md) 完全相同。当 source 为空时，Redis 将阻塞连接，直到另外一个客户端 push 元素到它，或等到 timeout 触发超时（指定要阻塞的最大秒数的 double 值）。当 timeout 为零时，会无限期的阻塞。此命令代替现在已弃用的 [BRPOPLPUSH](BRPOPLPUSH.md)。执行 `BLMOVE RIGHT LEFT` 是等效的。

有关详细信息，请参阅 [LMOVE](LMOVE.md)。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) :  元素从 source 中弹出，并压入 destination 中。如果达到 timeout 时限，会返回一个 [Null reply](../topics/protocol.md#null-bulk-string) 。

---

### Pattern: Reliable queue

请参阅 [LMOVE](LMOVE.md) 文档中的模式描述。

---

### Pattern: Circular list

请参阅 [LMOVE](LMOVE.md) 文档中的模式描述。