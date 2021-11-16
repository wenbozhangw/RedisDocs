## UNLINK key [key ...]

    起始版本：4.0.0.
    时间复杂度：对于删除的每个 key，无论其大小如何，都是 O(1)。然后该命令在不同的线程中执行 O(N) 工作以回收内存，其中 N 是组成已删除对象的分配数。

该命令和 [DEL](del.md) 十分相似：删除指定的 key。就像 [DEL](del.md) 一样，如果 key 不存在，则会被忽略。然而，该命令在不同的线程中执行实际的内存回收，因此它不会阻塞，而 [DEL](del.md) 会阻塞。这就是命令名称的来源：该命令只是从 keyspace 中**取消 key 的连接**。实际的删除将在稍后异步执行。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：取消链接的 key 的数量。

---

### Examples

```
redis> SET key1 "Hello"
"OK"
redis> SET key2 "World"
"OK"
redis> UNLINK key1 key2 key3
(integer) 2
redis> 
```