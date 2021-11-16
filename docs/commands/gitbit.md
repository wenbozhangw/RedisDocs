## GETBIT key offset

    起始版本：2.2.0.
    时间复杂度：O(1)。

返回 _key_ 对应的字符串在 _offset_ 处的 _bit_ 值。

当 _offset_ 超出了字符串长度的时候，这个字符串就被假定为由 0 bit 填充的连续空间。当 _key_ 不存在时，它就认为是一个空字符串，所以 _offset_ 总是超出范围，并且该值也被假定为具有 0 bit 的连续空间。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：存储在 _offset_ 的 bit 值。

---

### Examples

```
redis> SETBIT mykey 7 1
(integer) 0
redis> GETBIT mykey 0
(integer) 0
redis> GETBIT mykey 7
(integer) 1
redis> GETBIT mykey 100
(integer) 0
redis> 
```