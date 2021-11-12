## PTTL key

    起始版本：2.6.0.
    时间复杂度：O(1)。

这个命令类似于 [TTL](TTL.md) 命令，此命令返回设置了过期时间的 key 的剩余生存时间，唯一的区别是 [TTL](TTL.md) 以秒为单位返回剩余时间量，而 [PTTL](PTTL.md) 以毫秒为单位返回剩余时间。

在 Redis 2.6 或更早版本中，如果 key 不存在或者 key 存在但没有关联的过期时间，则该命令返回 **-1**。

从 Redis 2.8 开始，发送错误时的返回值发生了变化：
- 如果 key 不存在返回 **-2**。
- 如果 key 存在且无过期时间返回 **-1**。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：TTL 以毫秒为单位，或者返回一个错误值（参考上面的描述）。

---

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> EXPIRE mykey 1
(integer) 1
redis> PTTL mykey
(integer) 1000
redis> 
```