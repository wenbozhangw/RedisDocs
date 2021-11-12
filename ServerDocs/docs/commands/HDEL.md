## HDEL key field [field ...]

    起始版本：2.0.0.
    时间复杂度：O(N)，其中 N 为被移除的 fields 数量。

从 key 对应的 hash 中移除指定的 fields。在 hash 中不存在的 field 将被忽略。如果 key 对应的 hash 不存在，他将被认为是一个空的 hash，该命令将返回 0.

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：返回从 hash 中成功移除的 fields 的数量，不包括不存在的 fields。

---

### History

- &gt;= 2.4：可接收多个 fields 作为参数。小于 2.4 版本的 Redis 每次调用只能移除一个 field。

要在早期版本中以原子方式从 hash 中移除多个 field，可以使用 [MULTI](MULTI.md)/[EXEC](EXEC.md) 块。

---

### Examples

```
redis> HSET myhash field1 "foo"
(integer) 1
redis> HDEL myhash field1
(integer) 1
redis> HDEL myhash field2
(integer) 0
redis> 
```