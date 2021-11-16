## SINTERSTORE destination key [key ...]

    起始版本：1.0.0.
    时间复杂度：O(N * M) 最坏情况，其中 N 是最小集合的基数，M 是集合的数量。

此命令等于 [SINTER](sinter.md)，但它不会返回结果集，而是存储在目标中。

如果 `destionation` 已存在，它会被覆盖。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：结果集中成员的个数。

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
redis> SINTERSTORE key key1 key2
(integer) 1
redis> SMEMBERS key
1) "c"
redis> 
```