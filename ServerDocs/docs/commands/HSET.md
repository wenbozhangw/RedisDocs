## HSET key field value [field value ...]

    起始版本：2.0.0。
    时间复杂度：对于添加的每个 field/value pair，O(1)，因此当使用多个 field/value pair 调用命令时，添加 N 个 field/value pair，为 O(N)。

设置 key 指定的 hash set 中指定字段的值。如果 key 指定的 hash set 不存在，会创建一个新的 hash set 并与 key 关联。如果字段在 hash set 中存在，他将被重写。

从 Redis 4.0.0 开始，HSET 是可变参数并允许多个field/value pair。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 添加的字段数量。

---

### Examples

```
redis> HSET myhash field1 "Hello"
(integer) 1
redis> HGET myhash field1
"Hello"
redis> 
```