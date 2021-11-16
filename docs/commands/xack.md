## XACK key group ID [ID ...]

    起始版本：5.0.0.
    时间复杂度：O(1)，对于每条消息 ID 处理。

[XACK](xack.md) 命令用于从 Stream 的消费者组的 _Pending Entries List(PEL)_ 中删除一条或多条消息。当一条消息交付到某个消费者时，他将被存储在 PEL 中等待处理，这通常出现在作为调用 [XREADGROUP](xreadgroup.md) 命令的副作用，或者一个消费者通过调用 [XCLAIM](xclaim.md) 命令接管消息的时候。待处理消息被交付到某些消费者，但是服务器尚不确定他是否至少被处理了一次。因此对新调用 [XREADGROUP](xreadgroup.md) 来获取消费者的消息历史记录（比如用 0 作为 ID）将返回此类消息。类似地，待处理的消息将由检查 PEL 的 [XPENDING](xpending.md) 命令列出。

一旦消费者 _成功地(successfully)_ 处理完一条消息，他应该调用 [XACK](xack.md)，这样这个消息就会不被再次处理，且作为一个副作用，关于此消息的 PEL 条目也会被清除，从 Redis 服务器释放内存。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：

该命令返回成功确认的消息数。某些消息 ID 可能不再是 PEL 的一部分（例如因为它们已经被确认），而且 XACK 不会把他们算到成功确认的数量中。

---

### Examples

```
redis> XACK mystream mygroup 1526569495631-0
(integer) 1
```