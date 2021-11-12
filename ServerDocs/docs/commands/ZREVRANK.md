## ZREVRANK key member

    起始版本：2.0.0.
    时间复杂度：O(log(N))

返回存储在 key 对应的 Sorted Set 中 `member` 的排名(`rank`)，分数从高到低排序。排名（或索引）从 0 开始，这意味着得分最低的成员的排名为 0。

使用 [ZREVRANK](ZREVRANK.md) 获取分数从低到高排序的元素的排名。

---

### Return Value

- 如果 Sorted Set 存在 `member`，[Integer reply](../topics/protocol.md#resp-integers)：`rank` of `member`.
- 如果 Sorted Set 中不存在成员或 key 不存在， [Bulk string reply](../topics/protocol.md#resp-bulk-strings)：nil.

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZREVRANK myzset "one"
(integer) 2
redis> ZREVRANK myzset "four"
(nil)
redis> 
```