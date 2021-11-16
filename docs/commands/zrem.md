## ZREM key member [member ...]

    起始版本：1.2.0.
    时间复杂度：O(M * log(N))，其中 N 是 sorted set 中的元素数量， M 是要删除的元素数量。

从存储在 key 的 sorted set 中删除指定的成员。不存在的成员将被忽略。

当 key 存在，但是其不是 sorted set 类型，就返回一个错误。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：

- 返回的是从 sorted set 中删除的成员个数，不包括不存在的成员。

---

### History

- &gt;= 2.4：接受多个元素。在 2.4 之前的版本中，每次只能删除一个成员。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZREM myzset "two"
(integer) 1
redis> ZRANGE myzset 0 -1 WITHSCORES
1) "one"
2) "1"
3) "three"
4) "3"
redis> 
```