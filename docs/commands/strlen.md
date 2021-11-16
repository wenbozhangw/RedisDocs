## STRLEN key

    起始版本：2.2.0.
    时间复杂度：O(1)

返回 key 对应的字符串类型的 value 的长度。如果 key 对应的是非字符串类型，就返回错误。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：key 对应的字符串 value 的长度；如果 key 不存在，则为 0。

---

### Examples

```
redis> SET mykey "Hello world"
"OK"
redis> STRLEN mykey
(integer) 11
redis> STRLEN nonexisting
(integer) 0
redis> 
```