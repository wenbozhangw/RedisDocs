## ZREVRANGEBYLEX key max min [LIMIT offset count]

    起始版本：2.8.9.
    时间复杂度：O(log(N) + M)，其中 N 是 Sorted Set 中的元素数，M 是返回的元素数。如果 M 是常数（例如，总是用 `LIMIT` 要求前 10 个元素），你可以认为它是 O(log(N))。

当 Sorted Set 中的所有元素都以相同的分数插入时，为了强制按照字典顺序排列，此命令返回 key 对应的 Sorted Set 中的所有在 `<min>` 和 `<max>` 之间的元素。

除了顺序相反之外，[ZREVRANGEBYLEX](ZREVRANGEBYLEX.md) 与 [ZRANGEBYLEX](ZRANGEBYLEX.md) 相似。

根据 Redis 6.2.0，此命令被视为已弃用。请在新代码中使用带有 `BYLEX` 和 `REV` 参数的 [ZRANGE](ZRANGE.md) 命令。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：指定分数范围内的元素列表。

---

### Examples

```
redis> ZADD myzset 0 a 0 b 0 c 0 d 0 e 0 f 0 g
(integer) 7
redis> ZREVRANGEBYLEX myzset [c -
1) "c"
2) "b"
3) "a"
redis> ZREVRANGEBYLEX myzset (c -
1) "b"
2) "a"
redis> ZREVRANGEBYLEX myzset (g [aaa
1) "f"
2) "e"
3) "d"
4) "c"
5) "b"
redis> 
```