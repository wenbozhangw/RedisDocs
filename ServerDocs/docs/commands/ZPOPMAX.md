## ZPOPMAX key [count]

    起始版本：5.0.0.
    时间复杂度：O(log(N) * M) ，其中 N 是 sorted set 中的元素数， M 是弹出的元素数。

删除并返回指定 key 对应的 sorted set 中的最多 `count` 个具有最高得分的成员。

如未指定，`count` 的默认值为 1。指定一个大于有序集合的基数的 `count` 不会产生错误。当返回多个元素的时候，得分最高的元素将是第一个元素，然后是分数较低的元素。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) ：弹出的元素和分数列表。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZPOPMAX myzset
1) "three"
2) "3"
redis> 
```