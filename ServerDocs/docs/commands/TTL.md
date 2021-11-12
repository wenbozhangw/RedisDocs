## TTL key

    起始版本：1.0.0.
    时间复杂度：O(1)。

返回具有超时的 key 的剩余生存时间。这种内省功能允许 Redis 客户端检查给定 key 将继续成为数据集一部分的秒数。

在 Redis 2.6 或更早版本中，如果 key 不存在或者 key 存在但没有关联的过期时间，则该命令返回 **-1**。

从 Redis 2.8 开始，发送错误时的返回值发生了变化：
- 如果 key 不存在返回 **-2**。
- 如果 key 存在且无过期时间返回 **-1**。

另请参阅以毫秒单位返回相同信息的 [PTTL](PTTL.md) 命令。（仅适用于 Redis 2.6 或更高版本）。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：TTL 以秒为单位，或者返回一个负数错误值（参考上面的描述）。

---

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> EXPIRE mykey 10
(integer) 1
redis> TTL mykey
(integer) 10
redis> 
```