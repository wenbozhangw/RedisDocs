## RENAME key newkey

    起始版本：1.0.0。
    时间复杂度：O(1)。

将 key 重命名为 newkey。当 key 不存在时，返回错误。如果 newkey 已经存在，则值将会被覆盖，当这种情况发生时，[RENAME](RENAME.md) 会执行一个隐式的 [DEL](DEL.md) 操作，所以如果被删除的 key 包含一个非常大的值，即使 [RENAME](RENAME.md) 本身通常是一个常数时间操作，它也可能导致高延迟。

在集群模式下，key 和 newkey 必须在同一个**哈希槽(hash slot)**中，这意味着实际上只有具有相同哈希标签的 key 才能在集群中可靠地重命名。

---

### History

- &gt;= 3.2.0：在Redis 3.2.0之前，如果源名称和目标名称相同，则会返回错误。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)

---

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> RENAME mykey myotherkey
"OK"
redis> GET myotherkey
"Hello"
redis> 
```