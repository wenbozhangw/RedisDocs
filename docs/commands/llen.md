## LLEN key

    起始版本：1.0.0.
    时间复杂度：O(1)。

返回存储在 key 中的列表的长度。如果 key 不存在，则将其作为空列表并返回 0.当 key 中存储的值不是列表时，会返回错误。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：key 对应的列表长度。

---

### Examples

```
redis> LPUSH mylist "World"
(integer) 1
redis> LPUSH mylist "Hello"
(integer) 2
redis> LLEN mylist
(integer) 2
```