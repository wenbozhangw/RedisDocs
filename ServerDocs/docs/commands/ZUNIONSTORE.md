## ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]

    起始版本：2.0.0.
    时间复杂度：O(N) + O(M log(M))，其中 N 是输入 sorted set 的大小之和，M 是结果 sorted set 中的元素数。

计算给定的 `numkeys` 个 sorted set 的并集，并把结果放到 `detination` 中。在给定要计算的 key 和其他参数（可选）之前，必须先给定 key 的个数（`numkeys`）。

默认情况下，结果集中某个成员的 score 值是所有给定集合中该成员 score 值之和。

使用 `WEIGHTS` 选项，你可以为每个给定的 sorted set 指定一个乘法因子，意思就是，每个给定 sorted set 的所有成员的 score 值在传递给聚合函数之前都要先乘以该因子。如果 `WEIGHTS` 没有给定，默认就是 1。

使用 `AGGREGATE` 选项，你可以指定并集的结果集的聚合方式。默认使用的参数 `SUM`，可以将所有集合中某个成员的 score 值之和作为结果集中该成员的 score 值。如果使用参数 `MIN` 或者 `MAX`，结果集就是所有集合中元素的最小或最大的元素。

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
redis> ZUNIONSTORE out 2 zset1 zset2 WEIGHTS 2 3
(integer) 3
redis> ZRANGE out 0 -1 WITHSCORES
1) "one"
2) "5"
3) "three"
4) "9"
5) "two"
6) "10"
redis> 
```