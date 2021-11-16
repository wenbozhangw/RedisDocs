## INCR key

    起始版本：1.0.0。
    时间复杂度：O(1)。

对存储在指定 key 的数值执行原子的加一操作。如果指定的 key 不存在，那么在执行 incr 操作之前，会先将它的值设定为 0。如果指定的 key 包含错误的类型的值或包含不能表示为整数的字符串，则返回错误。此操作仅限于 64 位有符号整数。

**注意**：这是一个字符串操作，因为 Redis 没有专用的整数类型。存储在 key 中的字符串被解析为 **10进制的64位有符号整数** 来执行操作。

事实上，Redis 内部采用整数形式（Integer representation）来存储对应的整数值，所以对该类字符串值实际上是用整数保存，也就不存在存储整数的字符串表示（String representation）所带来的额外消耗。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 执行递增操作后 key 对应的值。

---

### Examples

```
redis> SET mykey "10"
"OK"
redis> INCR mykey
(integer) 11
redis> GET mykey
"11"
redis> 
```

---

### Pattern: Counter

Redis 的原子递增操作最常见地使用场景是计数器。使用思路是：每次有相关操作的时候，就像 Redis 服务器发送一个 [INCR](incr.md) 命令。例如这样一个场景：我们有一个 web 应用，我们想记录该用户一年中每天访问这个网站的次数。

为此，Web 应用程序可以在每次用户执行页面查看时简单地执行 incr 命令，其 key 可以将用户 ID 和一个标识当前日期的字符串连接起来。

这个场景可以有很多种扩展方法：

- 通过结合使用 [INCR](incr.md) 和 [EXPIRE](expire.md) 命令，可以实现一个只记录用户在指定间隔时间内的访问次数的计数器。
- 客户端可以通过 [GETSET](getset.md) 命令获取当前计数器的值并且重置为 0。
- 通过类似于 [DECR]() 或者 [INCRBY]() 等原子递增/递减命令，可以根据用户的操作来增加或减少某些值，比如说在线游戏，需要对用户的游戏分数进行实时控制，分数可能增加也可能减少。

---

### Pattern : Rate limiter

限速器是一种可以限制某些操作执行速率的特殊场景。传统的例子就是限制某个公共 api 的请求数目。

我们使用 [INCR](incr.md) 命令提供了这种模式的两种实现，假如我们要解决如下问题：限制某个 api 每秒每个 ip 地址的请求不超过 10 次。

#### Pattern : Rate limiter1

更加简单和直接的实现如下：
```
FUNCTION LIMIT_API_CALL(ip)
ts = CURRENT_UNIX_TIME()
keyname = ip+":"+ts
MULTI
    INCR(keyname)
    EXPIRE(keyname,10)
EXEC
current = RESPONSE_OF_INCR_WITHIN_MULTI
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    PERFORM_API_CALL()
END
```

这种方法的基本点是每个 ip 每秒生成一个可以记录请求的计数器。但是这些计数器每次递增的时候都设置了 10 s 的过期时间，这样在进入下一秒之后，过一段时间，会被redis自动删除。

注意上面伪代码中我们用到了 [MULTI](multi.md) 和 [EXEC](exec.md) 命令，将递增操作和设置过期时间操作的操作放在了一个事务中，从而保证了两个操作的原子性。

#### Pattern : Rate limiter2

另外一个实现是对每个 ip 只用一个单独的计数器（不是每秒生成一个），但是需要注意避免竟态条件。我们会对多种不同的变量进行测试。

```
FUNCTION LIMIT_API_CALL(ip):
current = GET(ip)
IF current != NULL AND current > 10 THEN
    ERROR "too many requests per second"
ELSE
    value = INCR(ip)
    IF value == 1 THEN
        EXPIRE(ip,1)
    END
    PERFORM_API_CALL()
END
```

上述方法的思路是，从第一个请求开始设置过期时间为 1 秒，如果同一秒内有超过 10 个请求，计数器将达到大于 10 的值，并返回错误，否则它将过期并重新从 0 开始。

**在上面的代码中有一个竟态条件**。如果由于某种原因客户端执行了 [INCR](incr.md) 命令，但是没用执行 [EXPIRE](expire.md)，则 key 将会造成内存泄露，直到下次有同一个 ip 发送请求过来。

这可以很容易地修复，将带有可选 [EXPIRE](expire.md) 的 [INCR](incr.md) 转换为使用 [EVAL](eval.md) 命令发送的 Lua 脚本（仅自 Redis 2.6 版起可用）。

```lua
local current
current = redis.call("incr",KEYS[1])
if current == 1 then
    redis.call("expire",KEYS[1],1)
end
```

有一种不同的方法可以在不使用脚本的情况下解决此问题，使用 Redis list而不是计数器。实现更复杂并使用更高级的功能，但具有可以记录当前请求 API 调用的客户端 IP 地址的优点，这是否有用取决于应用程序本身。

```
FUNCTION LIMIT_API_CALL(ip)
current = LLEN(ip)
IF current > 10 THEN
    ERROR "too many requests per second"
ELSE
    IF EXISTS(ip) == FALSE
        MULTI
            RPUSH(ip,ip)
            EXPIRE(ip,1)
        EXEC
    ELSE
        RPUSHX(ip,ip)
    END
    PERFORM_API_CALL()
END
```

如果 key 存在的话，[RPUSHX](rpushx.md) 命令会向 list 中插入一个元素。

请注意，我们在这里有一个竟态条件，但这不是问题：[EXISTS](exists.md) 命令可能返回 false，但在 [MULTI](multi.md)/[EXEC](exec.md) 块之前，另一个客户端创建了这个 key。后果就是我们会少记录一个请求。但是这种情况很少出现，所以我们的请求限速器还是能运行良好。