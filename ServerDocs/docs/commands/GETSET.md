## GETSET key value

    起始版本：1.0.0
    时间复杂度：O(1)。

原子地将 key 设置为对应的 value，并返回 key 之前存储的旧 value。如果 key 存在但是不包含字符串值时，返回错误。成功的 [SET](SET.md) 操作将丢弃与 key 关联的任何先前的过期时间。

---

### Design pattern

[GETSET](GETSET.md) 可以与 [INCR](INCR.md) 一起使用实现支持重置的计数功能。举个例子：每当有事件发生的时候，一段程序都会调用 [INCR](INCR.md) 给 key `mycounter` 加一，但是有时候我们需要获取计数器的值，并且自动将其重置为 0。这可以通过 `GETSET mycounter "0"` 来实现：

```
redis> INCR mycounter
(integer) 1
redis> GETSET mycounter "0"
"1"
redis> GET mycounter
"0"
redis> 
```

根据 Redis 6.2，GETSET 被视为已弃用。请在新代码中使用带有 [GET](GET.md) 参数的 [SET](SET.md) 命令。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 返回之前的旧值，如果之前Key不存在将返回nil。

---

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> GETSET mykey "World"
"Hello"
redis> GET mykey
"World"
redis> 
```