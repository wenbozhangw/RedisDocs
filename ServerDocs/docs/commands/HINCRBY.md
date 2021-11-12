## HINCRBY key field increment

    起始版本：2.0.0.
    时间复杂度：O(1)。

增加 key 对应的 hash 中指定 field 的值。如果 key 不存在，会创建一个新的 hash 并与 key 关联。如果 field 不存在，则字段的值在执行该操作前被设置为 0.

[HINCRBY](HINCRBY.md) 支持的值的范围限定在 64 为有符号整数。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：increment 操作执行后该 field 的值。

---

### Examples

由于 `increment` 参数是带符号的，因此可以执行 increment 和 decrement 操作：

```
redis> HSET myhash field 5
(integer) 1
redis> HINCRBY myhash field 1
(integer) 6
redis> HINCRBY myhash field -1
(integer) 5
redis> HINCRBY myhash field -10
(integer) -5
redis>
```