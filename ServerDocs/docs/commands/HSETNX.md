## HSETNX key field value

    起始版本：2.0.0.
    时间复杂度：O(1).

当且仅当 key 对应的 hash 中不存在 field 时，才设置字段的值。如果 key 对应的 hash 不存在，会创建一个新的 hash 并与 key 关联。如果 field 已经存在，该操作无效果。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- 1：如果字段是个新字段，并且成功赋值
- 0：如果 hash 中已存在该字段，没有操作成功

---

### Examples

```
redis> HSETNX myhash field "Hello"
(integer) 1
redis> HSETNX myhash field "World"
(integer) 0
redis> HGET myhash field
"Hello"
redis>
```