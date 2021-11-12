## ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]

    起始版本：2.0.0。
    时间复杂度：O(N * K) + O(M * log(M))，最后情况，N 是最小的输入 sorted set，K 是输入 sorted set 的数量，M 是结果 sorted set 中的元素数量。

计算给定的 `numkeys` 个 sorted set 的交集，并且把结果放到 `destination` 中。在给定要计算的 key 和其他（可选）参数之前，必须先给定 key 的个数(`numkeys`)。

默认情况下，结果中一个元素的分数是 sorted set 中该元素分数之和，前提是该元素在这些 sorted set 中都存在。因为交集要求其成员必须是给定的每个 sorted set 中的成员，结果集中的每个元素的分数和输入的 sorted set 个数相等。

对于 `WEIGHTS` 和 `AGGREGATE` 参数的描述，参见命令 [ZUNIONSTORE](ZUNIONSTORE.md)。

如果 `destination` 存在，就把它覆盖。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：结果 sorted set `destination` 中元素个数。

---

### Examples

```
redis> ZADD zset1 1 "one"
(integer) 1
redis> ZADD zset1 2 "two"
(integer) 1
redis> ZADD zset2 1 "one"
(integer) 1
redis> ZADD zset2 2 "two"
(integer) 1
redis> ZADD zset2 3 "three"
(integer) 1
redis> ZINTERSTORE out 2 zset1 zset2 WEIGHTS 2 3
(integer) 2
redis> ZRANGE out 0 -1 WITHSCORES
1) "one"
2) "5"
3) "two"
4) "10"
redis> 
```