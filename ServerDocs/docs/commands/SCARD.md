## SCARD key

    自 1.0.0 起可用。
    时间复杂度：O(1)。

返回集合存储的 key 的基数（集合元素的数量）。

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 集合的基数（元素的数量），如果 key 不存在，则返回 0。

### Examples

```
redis> SADD myset "Hello"
(integer) 1
redis> SADD myset "World"
(integer) 1
redis> SCARD myset
(integer) 2
redis>
```