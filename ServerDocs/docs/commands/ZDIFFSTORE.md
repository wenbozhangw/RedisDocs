## ZDIFFSTORE destination numkeys key [key ...]

    起始版本：6.2.0.
    时间复杂度：O(L + (N-K)log(N)) 最坏情况，其中 L 是所有集合汇总元素的总数，N 是第一个集合的大小，K 是结果集的大小。

计算第一个和所有连续输入 sorted set 之前的差异，并将结果存储在 `destination` 中。输入键的总数由 `numkeys` 指定。

不存在的 key 被认为是空集。

如果 `destination` 已经存在，会被覆盖。

---

### Return Value

[Integer value](../topics/protocol.md#resp-integers)：在 sorted set `destination` 中元素的数量。

---

### Examples

```
redis> ZADD zset1 1 "one"
(integer) 1
redis> ZADD zset1 2 "two"
(integer) 1
redis> ZADD zset1 3 "three"
(integer) 1
redis> ZADD zset2 1 "one"
(integer) 1
redis> ZADD zset2 2 "two"
(integer) 1
redis> ZDIFFSTORE out 2 zset1 zset2
(integer) 1
redis> ZRANGE out 0 -1 WITHSCORES
1) "three"
2) "3"
redis> 
```