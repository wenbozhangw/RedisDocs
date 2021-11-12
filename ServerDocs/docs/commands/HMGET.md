## HMGET key field [field ...]

    起始版本：2.0.0.
    时间复杂度：O(N)，其中 N 是请求的字段数量。

返回与存储在 key 对应的 hash 中的指定字段关联的值。

对于 hash 中不存在的每个字段，返回一个 nil 值。因为不存在的 key 被视为空 hash，所以对不存在的 key 运行 [HMGET](HMGET.md) 将返回一个 nil 值列表。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：与给定字段关联的值列表，按请求顺序排列。

---

### Examples

```
redis> HSET myhash field1 "Hello"
(integer) 1
redis> HSET myhash field2 "World"
(integer) 1
redis> HMGET myhash field1 field2 nofield
1) "Hello"
2) "World"
3) (nil)
redis> 
```