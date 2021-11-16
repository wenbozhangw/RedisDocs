## DECRBY key decrement

    起始版本：1.0.0.
    时间复杂度：O(1)。

将 key 对应的数字减少 `decrement`。如果 key 不存在，操作之前，key 会被设置为 0。如果 key 的 value 类型错误，或者是个不能表示为数字的字符串，就返回错误。这个操作最多支持 64 位有符号整型数字。

查看命令 [INCR](incr.md) 了解关于 increment/decrement  操作的额外信息。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：减少之后的 value 值。

---

### Examples

```
redis> SET mykey "10"
"OK"
redis> DECRBY mykey 3
(integer) 7
redis> 
```