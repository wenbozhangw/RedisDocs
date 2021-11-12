## HMSET key field value [field value ...]

    起始版本：2.0.0.
    时间复杂度：O(N)，其中 N 为需要设置的 field 个数。

设置 key 对应的 hash 中指定 field 的值。该命令将覆盖所有在 hash 中存在的字段。如果 key 对应的 hash 不存在，会创建一个新的 hash 并与 key 关联。

根据 Redis 4.0.0，HMSET 被认为是弃用的。请选择新代码中的 [HSET](HSET.md)。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)

---

### Examples

```
redis> HMSET myhash field1 "Hello" field2 "World"
"OK"
redis> HGET myhash field1
"Hello"
redis> HGET myhash field2
"World"
redis> 
```