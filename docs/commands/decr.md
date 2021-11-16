## DECR key

    起始版本：1.0.0。
    时间复杂度：O(1)。

将存储在 key 中的数字减一。如果 key 不存在，则在执行操作前将其设置为 0。如果 key 包含错误类型的值或包含不能表示为整数的字符串，则返回错误。此操作仅限于 **64位有符号整数**。

查看命令 [INCR](incr.md) 了解关于增减操作的额外信息。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：减小之后的 value

---

### Examples

```
redis> SET mykey "10"
"OK"
redis> DECR mykey
(integer) 9
redis> SET mykey "234293482390480948029348230948"
"OK"
redis> DECR mykey
ERR ERR value is not an integer or out of range
redis> 
```