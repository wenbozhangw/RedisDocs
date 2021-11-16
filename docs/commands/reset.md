## RESET

    起始版本：自 6.2 起可用。

此命令执行连接的服务器端上下文的完全重置，模拟断开连接和重新连接的效果。

当从常规客户端连接调动该命令时，它会执行以下操作：
- 如果存在，则丢弃当前的 [MULTI](multi.md) 事务块。
- 接触对连接锁监听的所有 key 的监听 (Unwatches all keys `WATCHed` by the connection.) 。
- 如果正在使用，禁用 [CLIENT TRACKING](client-tracking.md) 。
- 将连接设置为 [READWRITE](readwrite.md) 模式。
- 取消连接的 [ASKING](asking.md) 模式（如果之前设置过）。
- 将 [CLIENT REPLY](client-reply.md) 设置为 `ON`。
- 将协议版本设置为 `RESP2`。
- `SELECT`s 数据库 0。
- 在适当的时候，退出 [MONITOR](monitor.md) 模式。
- 在适当的时候中止 Pub/Sub 的订阅模式（[SUBSCRIBE](subscribe.md) 和 [PSUBSCRIBE](psubscribe.md) ）。
- 取消验证连接，当启用验证时需要调用 [AUTH](auth.md) 重新验证。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：总是 'RESET' 。