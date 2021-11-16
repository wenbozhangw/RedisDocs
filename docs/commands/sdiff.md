## SDIFF key [key ...]

    起始版本：1.0.0.
    时间复杂度：O(N)，其中 N 为所有给定集合中元素的总数。

返回由第一个集合和所有后续集合之间的不同的成员。

例如：

```
key1 = {a,b,c,d}
key2 = {c}
key3 = {a,c,e}
SDIFF key1 key2 key3 = {b,d}
```

不存在的 key 被认为是空集。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：结果集的成员的列表。

---

### Examples

```
redis> SADD key1 "a"
(integer) 1
redis> SADD key1 "b"
(integer) 1
redis> SADD key1 "c"
(integer) 1
redis> SADD key2 "c"
(integer) 1
redis> SADD key2 "d"
(integer) 1
redis> SADD key2 "e"
(integer) 1
redis> SDIFF key1 key2
1) "b"
2) "a"
redis> 
```