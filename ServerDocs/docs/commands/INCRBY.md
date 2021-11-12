## INCRBY key increment

    起始版本：1.0.0。
    时间复杂度：O(1)。

将 key 对应的数字增加参数 `increment`。如果 key 不存在，则在执行操作前将其设置为 0。如果 key 包含错误类型的值或包含不能表示为整数的字符串，则返回错误。此操作仅限于 64 位有符号整数。有关 increment/decrement 操作的额外信息，请参阅 [INCR](INCR.md)。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 增加之后的 value 值。

---

### Examples

```
redis> SET mykey "10"
"OK"
redis> INCRBY mykey 5
(integer) 15
redis> 
```