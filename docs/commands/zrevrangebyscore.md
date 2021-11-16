## ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]

    起始版本：2.2.0.
    时间复杂度：O(log(N) + M)，其中 N 是 Sorted Set 中的元素数，M 是返回的元素数。如果 M 是常数（例如，总是用 `LIMIT` 要求前 10 个元素），你可以认为它是 O(log(N))。

返回 Sorted Set 中，分数在 `max` 和 `min` 之间的所有元素（包括相等的）。与 Sorted Set 的默认排序相反，对于此命令，元素会按照从高分到低分排序。

具有相同分数的元素以 reverse lexicographical 顺序返回。

除了顺序相反之外，[ZREVRANGEBYSCORE](zrevrangebyscore.md) 与 [ZRANGEBYSCORE](zrangebyscore.md) 相似。

根据 Redis 6.2.0，此命令被视为已弃用。请在新代码中使用带有 `BYSCORE` 和 `REV` 参数的 [ZRANGE](zrange.md) 命令。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：指定分数范围内的元素列表（可选及其分数）。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZREVRANGEBYSCORE myzset +inf -inf
1) "three"
2) "two"
3) "one"
redis> ZREVRANGEBYSCORE myzset 2 1
1) "two"
2) "one"
redis> ZREVRANGEBYSCORE myzset 2 (1
1) "two"
redis> ZREVRANGEBYSCORE myzset (2 (1
(empty list or set)
redis> 
```