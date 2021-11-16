## ZREMRANGEBYSCORE key min max

    起始版本：1.2.0
    时间复杂度：O(log(N)+M)，其中 N 是 Sorted set 中的元素数，M 是操作删除的元素数。

删除存储在 key 的 sorted set 中的所有元素，其分数介于 min 和 max 之间（包含）。

从 2.1.6 版本开始，min 和 max 可以是不包含的，遵循 [ZRANGEBYSCORE](zrangebyscore.md) 语法。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：删除的元素数量。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZREMRANGEBYSCORE myzset -inf (2
(integer) 1
redis> ZRANGE myzset 0 -1 WITHSCORES
1) "two"
2) "2"
3) "three"
4) "3"
redis> 
```