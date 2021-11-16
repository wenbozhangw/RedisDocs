## TYPE key

    起始版本：1.0.0。
    时间复杂度：O(1)。

返回存储在 key 中的 value 的数据结构类型。可以返回的类型有 : `string`, `list`, `set`, `zset`, `hash` 和 `stream`。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) : key 的类型，或者当 key 不存在时为无。

---

### Examples

```
redis> SET key1 "value"
"OK"
redis> LPUSH key2 "value"
(integer) 1
redis> SADD key3 "value"
(integer) 1
redis> TYPE key1
"string"
redis> TYPE key2
"list"
redis> TYPE key3
"set"
redis> 
```