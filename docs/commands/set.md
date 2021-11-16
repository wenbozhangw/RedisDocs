## SET key value [EX seconds | PX milliseconds | EXAT timestamp | PXAT milliseconds-timestamp | KEEPTTL] [NX|XX] [GET]

    起始版本：1.0.0。
    时间复杂度：O(1)。

将 key 设定为指定 "字符串" 值。如果 key 已经保存了一个值，那么这个操作会直接覆盖原来的值，并且忽略原始类型。当 [SET](set.md) 命令操作成功之后，之前设置的过期时间都将失效。

### Options

 [SET](set.md) 命令支持一组修改其行为的选项：
 - EX *seconds* —— 设置指定的过期时间，以秒为单位。
 - PX *milliseconds* —— 设置指定的过期时间，以毫秒为单位。
 - EXAT *timestamp-seconds* —— 设置指定 key 的 Unix 过期时间，以秒为单位。
 - PXAT *timestamp-milliseconds* —— 设置指定 key 的 Unix 过期时间，以毫秒为单位。
 - NX —— 仅在 key 不存在时才设置 key。
 - XX —— 仅在 key 存在时才设置 key。
 - KEEPTTL —— 保留与 key 关联的过期时间。
 - GET —— 返回存储在 key 中的旧字符串，如果 key 不存在，则返回 nil。如果 key 中存储的值不是字符串，则返回错误并中止 [SET](set.md) 。

注意：由于 SET 命令加上选项已经可以完全取代 [SETNX](setnx.md), [SETEX](setex.md), [PSETEX](psetex.md), [GETSET](getset.md) 的功能，所以在将来的版本中，redis 可能会不推荐使用并最终抛弃这几个命令。

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) : 如果 [SET](set.md) 执行正常，则返回 OK。

[Null reply](../topics/protocol.md#null-bulk-string) : (nil) 如果由于用户指定了 `NX` 或 `XX` 选项而未执行 [SET](set.md) 操作但不满足条件。

如果使用 [GET](get.md) 选项发出命令，则上述内容不适用。它将改为如下回复，无论 [SET](set.md) 是否实际执行：

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : key 中存储的旧的字符串值。
[Null reply](../topics/protocol.md#null-bulk-string) : (nil) 如果 key 不存在。

### History

- &gt;= 2.6.12 : 增加 EX, PX, NX 和 XX 选项。
- &gt;= 6.0 : 增加 KEEPTTL 选项。
- &gt;= 6.2 : 增加 GET, EXAT 和 PXAT 选项。
- &gt;= 7.0 : 允许一起使用 NX 和 GET 选项。

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> GET mykey
"Hello"
redis> SET anotherkey "will expire in a minute" EX 60
"OK"
redis> 
```

### Patterns

**注意** ： 下面这种设计模式并不推荐用来实现 redis 分布式锁。应该参考 [the Redlock algorithm](../topics/distlock.md) 的实现，因为这个算法实现起来稍微复杂一点，但提供了更好的保证并且具有容错性。

命令 `SET resource-name anystring NX EX max-lock-time` 是一种用 Redis 来实现锁机制的简单方法。

如果上述命令返回 OK ，那么客户端就可以获得锁（如果上述命令返回 Nil，那么客户端可以在一段时间之后重新尝试），并且可以通过 [DEL](del.md) 命令来释放锁。

客户端加锁之后，如果没有主动释放，会在过期时间之后自动释放。

可以通过如下优化使得上面的锁系统变得更加鲁棒：

- 不要设置固定的字符串，而是设置为随机的大字符串，也就是 token。
- 不要使用 [DEL](del.md) 释放锁，而是发送一个脚本，该脚本仅在值匹配时删除 key 。

上述方法会避免下述场景：a客户端获得的锁（key）已经由于过期时间到了被 redis 服务器删除，但是这个时间 a 客户端还去执行 [DEL](del.md) 命令。而 b 客户端已经在 a 设置的过期时间之后重新获取了这个同样 key 的锁，那么 a 执行 [DEL](del.md) 就会释放了 b 客户端加好的锁。

解锁脚本的一个例子如下：

```
if redis.call("get",KEYS[1]) == ARGV[1]
then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

这个脚本执行方式如下：

```
EVAL …script… 1 resource-name token-value
```