## HINCRBYFLOAT key field increment

    起始版本：2.6.0.
    时间复杂度：O(1)。

为指定的 `key` 对应的 hash 的 `field` value 加上 float 类型的 `increment`。如果 `increment` 值为负数，则结果是 hash field 减少而不是增加。如果该字段不存在，则在执行操作前将其设置为 0。如果出现以下情况之一，返回错误：
- `field` 的值包含的类型错误（不是字符串）。
- 当前 `field` 或者 `increment` 不能解析为一个 float 类型。

此命令的确切行为与 [INCRBYFLOAT](INCRBYFLOAT.md) 命令相同，请参阅 [INCRBYFLOAT](INCRBYFLOAT.md) 命令获取更多信息。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)：field 加上 increment 之后的值。

---

### Examples

```
redis> HSET mykey field 10.50
(integer) 1
redis> HINCRBYFLOAT mykey field 0.1
"10.6"
redis> HINCRBYFLOAT mykey field -5
"5.6"
redis> HSET mykey field 5.0e3
(integer) 0
redis> HINCRBYFLOAT mykey field 2.0e2
"5200"
redis> 
```

---

### Implementation details

该命令总是作为 [HSET](HSET.md) 操作在副本连接(replication link)和 Append Only 文件中传播，因此底层浮点数学实现中的差异不会成为不一致的来源。
