## SINTER key [key ...]

    起始版本：1.0.0。
    时间复杂度：最坏情况 O(N*M)，其中 N 是最小 set 的基数(cardinality)，M 是 set 的数量。

返回指定所有的集合的成员的交集。

例如：

```
key1 = {a,b,c,d}
key2 = {c}
key3 = {a,c,e}
SINTER key1 key2 key3 = {c}
```

不存在的 key 被认为是空集合。由于其中一个 key 是空集合，所以结果集也是空的（因为任何集合与空集合的交集总是空集合）。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) : 列出结果集的成员。

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
redis> SINTER key1 key2
1) "c"
redis> 
```