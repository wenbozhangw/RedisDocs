## GET key

    起始版本：1.0.0。
    时间复杂度：O(1)。

返回 key 的 value。如果 key 不存在，返回特殊值 nil。如果 key 的 value 不是 string，就返回错误，因为 [GET](GET.md) 只处理 string 类型的值。

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : key 对应的 value，或者 nil（key 不存在时）。

### Examples

```
redis> GET nonexisting
(nil)
redis> SET mykey "Hello"
"OK"
redis> GET mykey
"Hello"
redis> 
```