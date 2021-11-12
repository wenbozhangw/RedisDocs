## ZREMRANGEBYLEX key min max

    起始版本：2.8.9.
    时间复杂度：O(log(N) + M)，其中 N 是 Sorted set 中的元素数，M 是操作删除的元素数。

当 sorted set 中的所有元素都以相同的分数插入时，为了强制按 lexicographical 顺序排列，此命令删除存储在由 min 和 max 指定的 lexicographical 范围之间的指定 key 的 sorted set 中所有的元素。

min 和 max 的含义与 [ZRANGEBYLEX](ZRANGEBYLEX.md) 命令相同。类似的，如果使用相同的 min 和 max 参数调用 [ZRANGEBYLEX](ZRANGEBYLEX.md) 将返回相同的元素，该命令实际上会删除这些元素。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：删除的元素数量。

---

### Examples

```
redis> ZADD myzset 0 aaaa 0 b 0 c 0 d 0 e
(integer) 5
redis> ZADD myzset 0 foo 0 zap 0 zip 0 ALPHA 0 alpha
(integer) 5
redis> ZRANGE myzset 0 -1
1) "ALPHA"
 2) "aaaa"
 3) "alpha"
 4) "b"
 5) "c"
 6) "d"
 7) "e"
 8) "foo"
 9) "zap"
10) "zip"
redis> ZREMRANGEBYLEX myzset [alpha [omega
(integer) 6
redis> ZRANGE myzset 0 -1
1) "ALPHA"
2) "aaaa"
3) "zap"
4) "zip"
redis> 
```