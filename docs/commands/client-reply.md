## CLIENT REPLY ON|OFF|SKIP

    起始版本：3.2.0.
    时间复杂度：O(1).

当需要完全禁用 Redis 服务器对当前客户端的回复时可使用该命令。

例如，当客户端发送 fire 和 forget 命令或执行大量数据加载时；或正在建立缓存，数据在不断传输过程中，客户端会忽略收到的回复，此时消耗服务器时间和带宽回复客户端，是一种资源浪费。

注：fire 和 forget 就是发送命令，然后完全不关心最终什么时候完成命令操作。

[CLIENT REPLY](client-reply.md) 可设置服务器是否对客户端的命令进行回复。有如下选项：

- `ON`。默认选项，不回复客户端命令。
- `OFF`。不回复客户端命令。
- `SKIP`。跳过该命令的回复。

---

### Return Value

当执行命令设置 `OFF` 或 `SKIP`，设置命令收不到任何回复，当设置为 `ON` 时，返回 OK 。

[Simple string reply](../topics/protocol.md#resp-simple-strings)：OK。