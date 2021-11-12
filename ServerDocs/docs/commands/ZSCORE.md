## ZSCORE key member

    起始版本：1.2.0.
    时间复杂度：O(1)。

返回 key 对应的 sorted set 中，成员 member 的 score 值。

如果 `member` 元素不是 key 对应的 sorted set 的成员，或者 key 不存在，返回 nil。 

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)：member 成员的 score 值（double 类型浮点数），以字符串形式表示。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZSCORE myzset "one"
"1"
redis> 
```