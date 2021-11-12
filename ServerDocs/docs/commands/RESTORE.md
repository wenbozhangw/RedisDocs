## RESTORE key ttl serialized-value [REPLACE] [ABSTTL] [IDLETIME seconds] [FREQ frequency]

    起始版本：2.6.0.
    时间复杂度：O(1) 来创建新 key 和额外的 O(N*M) 来重建序列化值，其中 N 是组成该值的 Redis 对象的数量，M 是它们的平均大小。对于小字符串值，时间复杂度为 O(1) + O(1*M)，其中 M 很小，所以简化为 O(1)。然而，对于 sorted set 值，复杂度是 O(N*M*log(N))，因为将值插入到 sorted set 中是 O(log(N))。

反序列化给定的序列化值(通过 [DUMP](DUMP.md) 获得)，并将它和给定的 key 关联。

如果 ttl 为 0，则创建的 key 不会过期，否则设置指定的过期时间（以毫秒为单位）。

如果使用了 `ABSTTL` 修饰符，则 ttl 应表示为 key 将过期的绝对 [Unix timestamp](http://en.wikipedia.org/wiki/Unix_time) （以毫秒为单位）。（Redis 5.0 或更高版本）。

处于逐出目的，您可以使用 `IDLETIME` 或 `FREQ` 修饰符。有关更多信息，请参阅 [OBJECT](OBJECT.md) (Redis 5.0 或更高版本)。

除非您使用 `REPLACE` 修饰符（Redis 3.0或更高版本），否则当 key 已存在时，[RESTORE](RESTORE.md) 将返回 "Target key name is busy" 错误。

[RESTORE](RESTORE.md) 检查 RDB 版本和数据校验和。如果它们不匹配，则返回错误。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：如果成功，命令返回 OK。

---

### Examples

```
redis> DEL mykey
0
redis> RESTORE mykey 0 "\n\x17\x17\x00\x00\x00\x12\x00\x00\x00\x03\x00\
                        x00\xc0\x01\x00\x04\xc0\x02\x00\x04\xc0\x03\x00\
                        xff\x04\x00u#<\xc0;.\xe9\xdd"
OK
redis> TYPE mykey
list
redis> LRANGE mykey 0 -1
1) "1"
2) "2"
3) "3"
```