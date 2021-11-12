## ZREVRANGE key start stop [WITHSCORES]

    起始版本：1.2.0.
    时间复杂度：O(log(N)+M)，其中 N 是 Sorted Set 中的元素数，M 是返回的元素数。

返回存储在 key 对应的 Sorted Set 中指定范围的元素。元素是根据分数从高到低排序的。降序 lexicographical 顺序用于具有相同分数的元素。

除了顺序相反之外，[ZREVRANGE](ZREVRANGE.md) 与 [ZRANGE](ZRANGE.md) 相似。

根据 Redis 6.2.0，此命令被视为已弃用。请在新代码中使用带有 `REV` 参数的 [ZRANGE](ZRANGE.md) 命令。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：指定分数范围的元素列表（分数可选）。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZREVRANGE myzset 0 -1
1) "three"
2) "two"
3) "one"
redis> ZREVRANGE myzset 2 3
1) "one"
redis> ZREVRANGE myzset -2 -1
1) "two"
2) "one"
redis> 
```