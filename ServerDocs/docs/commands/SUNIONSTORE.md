## SUNIONSTORE destination key [key ...]

    起始版本：1.0.0.
    时间复杂度：O(N)，其中 N 为所有给定集合中元素的总数。

此命令与 [SUNION](SUNION.md) 相同，但不是返回结果集，而是存储在 `destination` 中。

如果 `destination` 已经存在，它会被覆盖。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：结果集中的元素数。

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
redis> SUNIONSTORE key key1 key2
(integer) 5
redis> SMEMBERS key
1) "b"
2) "c"
3) "e"
4) "a"
5) "d"
redis> 
```