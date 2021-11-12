## HGETALL key

    起始版本：1.0.0。
    时间复杂度：O(N)，其中 N 为 hash 的大小。

返回存储在 key 中的 hash 的所有字段和值。在返回的值中，每个字段名后跟着值，因此回复的长度是 hash 大小的两倍。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) : 存储在 hash 中的字段及其值的列表，或者当 key 不存在时为空列表。

---

### Examples

```
redis> HSET myhash field1 "Hello"
(integer) 1
redis> HSET myhash field2 "World"
(integer) 1
redis> HGETALL myhash
1) "field1"
2) "Hello"
3) "field2"
4) "World"
redis> 
```