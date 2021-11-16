## SMOVE source destination member

    起始版本：1.0.0
    时间复杂度：O(1)。

将成员从 `source` 集合移动到 `destination` 集合。这个操作是原子的。在每个给定的时刻，该元素对于其他客户端来说，只会作为 `source` 或 `destination` 集合的成员出现。

如果 `source` 集合不存在或不包含指定元素，则不执行任何操作并返回 0。否则，元素将从 `source` 集合中删除并添加到 `destination` 集合中。当指定的元素已经存在于 `destination` 集合中时，它只会从 `source` 集合中移除。

如果 `source` 或 `destination` 不是集合类型，则返回错误。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的:

- 如果该元素成功移除，返回 1
- 如果该元素不是 `source` 的成员并未执行任何操作，则为 0。

---

### Examples

```
redis> SADD myset "one"
(integer) 1
redis> SADD myset "two"
(integer) 1
redis> SADD myotherset "three"
(integer) 1
redis> SMOVE myset myotherset "two"
(integer) 1
redis> SMEMBERS myset
1) "one"
redis> SMEMBERS myotherset
1) "three"
2) "two"
redis> 
```