## HGET key field

    起始版本：2.0.0.
    时间复杂度：O(1)。

    返回 key 指定的 hash set 中该字段所关联的值。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)：该字段所关联的值。当字段不存在或者 key 不存在时返回 nil。

---

### Examples

```
redis> HSET myhash field1 "foo"
(integer) 1
redis> HGET myhash field1
"foo"
redis> HGET myhash field2
(nil)
redis> 
```