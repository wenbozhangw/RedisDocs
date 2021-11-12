## DUMP key

    起始版本：2.6.0.
    时间复杂度：访问 key 为O(1)，序列化它为 O(N*M)，其中 N 是组成值的 Redis 对象的数量，M 是它们的平均大小。对于小字符串值，时间复杂度为 O(1) + O(1*M)，其中 M 很小，所以简化为 O(1)。

以特定于 Redis 的格式序列化存储在 key 中的值并将其返回给用户。可以使用 [RESTORE](RESTORE.md) 命令将返回的值合成回 Redis 键。

序列化格式是不透明(opaque)和非标准的(non-standard)，但它有一些语义特征：
- 它带有 64 位的校验和，用于检测错误，[RESTORE](RESTORE.md) 在进行反序列化之前会先检查校验和。
- 值的编码格式和 RDB 文件保持一致。
- RDB 版本会被编码在序列化值当中，如果因为 Redis 的版本不同造成 RDB 格式不兼容，那么 Redis 会拒绝这个值进行反序列化的操作。

序列化值不包含过期信息。为了捕获当前值的生存时间，应该使用 [PTTL](PTTL.md) 命令。

如果 key 不存在，则返回 nil bulk reply。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)： 序列化值。

---

### Examples

```
redis> SET mykey 10
"OK"
redis> DUMP mykey
"\u0000\xC0\n\t\u0000\xBEm\u0006\x89Z(\u0000\n"
redis> 
```