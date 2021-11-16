## SUNION key [key ...]

    起始版本：1.0.0.
    时间复杂度：O(N)，其中 N 为所有给定的集合中元素总数。

返回由所有给定集合的并集产生的集合成员。

例如：

```
key1 = {a,b,c,d}
key2 = {c}
key3 = {a,c,e}
SUNION key1 key2 key3 = {a,b,c,d,e}
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
redis> SUNION key1 key2
1) "b"
2) "c"
3) "e"
4) "a"
5) "d"
redis> 
```