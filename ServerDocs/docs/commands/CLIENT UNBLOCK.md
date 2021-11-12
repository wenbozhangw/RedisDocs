## CLIENT UNBLOCK client-id [TIMEOUT | ERROR]

    自 5.0.0 起可用。
    时间复杂度：O(log N) 其中 N 是客户端连接数。

当客户端因为执行具有阻塞功能的命令（例如 [BRPOP](BRPOP.md)、[XREAD](XREAD.md)、[WAIT](wait.md) ）被阻塞时，该命令可以通过其他连接解除客户端的阻塞

默认情况下，当命令 timeout 设置超时时，客户端会被解除阻塞。但是如果传递了一个额外的（可选的）参数，则可以指定解除阻塞的行为，可以是 **TIMEOUT** 或 **ERROR**。如果指定了 **ERROR**，则行为是解除对客户端阻塞，客户端阻塞被强制解除同时收到如下明确报错信息。具体客户端会收到如下错误：

```
-UNBLOCKED client unblocked via CLIENT UNBLOCK
```

注意：当然，通常不能保证错误文本保持不变，但错误代码将可以保证 -UNBLOCKED。

这个命令在当我们用有限数量的连接，监听许多 key 时特别有用。例如，我们可能希望在不使用超过 N 个连接的情况下使用 [XREAD](XREAD.md) 监听多个消息流。然而，在某一时刻，信息流消费者进程被告知还有一个新的 stream key 需要监控。为了避免使用更多连接，最好的行为从连接池中停掉一个阻塞的连接，添加新地要监控的 key ，然后再次发出阻塞命令。

要获得此行为，请使用以下模式。该进程使用额外的 _control connection_，以便在需要时发送 [CLIENT UNBLOCK](CLIENT UNBLOCK.md) 命令。同时，在堆其他连接运行阻塞操作之前，该进程会运行 [CLIENT ID](CLIENT ID.md) 以获取该连接关联的 ID。当应添加新的 key 或不再监听 key 时，通过在控制连接中发送 [CLIENT UNBLOCK](CLIENT UNBLOCK.md) 来中止相关的连接阻塞命令。阻塞命令将返回并最终可以重新发出。

此示例显示了 Redis 流上下文中的应用程序，但是该模式是通用的，可以应用于其他情况。

```
Connection A (blocking connection):
> CLIENT ID
2934
> BRPOP key1 key2 key3 0
(client is blocked)

... Now we want to add a new key ...

Connection B (control connection):
> CLIENT UNBLOCK 2934
1

Connection A (blocking connection):
... BRPOP reply with timeout ...
NULL
> BRPOP key1 key2 key3 key4 0
(client is blocked again)
```

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) ，特殊的：

- 1，如果客户端解除阻塞成功。
- 0，如果客户端没有解除阻塞。