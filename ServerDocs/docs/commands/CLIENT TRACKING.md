## CLIENT TRACKING ON|OFF [RED|RECT client-id] [PREFIX prefix [PREFIX prefix ...]] [BCAST] [OPTION] [OPTOUT] [NOLOOP]

    自 6.0.0 起可用。
    时间复杂度：O(1)。某些选项可能会引入额外的复杂性。

此命令启用 Redis 服务器的跟踪功能，用于 [server assisted client side caching](../topics/client-side-caching.md) 。

启用跟踪后，Redis 会记住连接请求的 key，以便在修改这些 key 时发送失效消息 (invalidation messages)。失效消息在同一连接中发送（仅在使用 RESP3 协议时可用）或在不同连接中重定向（也可用于 RESP2 和 Pub/Sub）。指定为 _broadcasting_ 模式是可用的，在这种模式下，不管它们请求的是什么 key，参与此协议的客户端会收到每个订阅指定 key 前缀的通知。 鉴于参数的复杂性，请参阅 [this main client side caching documentation](../topics/client-side-caching.md) 了解详细信息。本手册页仅作为此子命令选项的参考。

要启用跟踪，请使用：
```
CLIENT TRACKING on ... options ...
```

该功能将在其整个生命周期中当前连接中保持活动状态，除非在某些时候打开跟踪并关闭 `CLIENT TRACKING` 。

以下是在启用跟踪时修改命令行为的选项列表：

- `REDIRECT <id>` : 向具有指定 ID 的连接发送 invalidation message。连接必须存在。您可以使用 [CLIENT ID](CLIENT ID.md) 获取连接的 ID。如果我们重定向到的连接被终止，当处于 RESP3 模式时，启用跟踪的连接将接收 *tracking-redir-broken* 推送消息，以便通知条件。
- `BCAST` ： 在广播模式下启用跟踪。在此模式下，会针对所有指定的前缀报告 invalidation message，而不管连接请求的 key 如何。相反，当广播模式未启用时，Redis 将使用只读命令跟踪获取了哪些 key，并且只会报告此类 key 的失效消息。
- `PREFIX <prefix>` ：对于广播，注册给定的 key 前缀，以便仅针对以该字符串开通的 key 提供通知。可以多次给出此选项以注册多个前缀。如果在没有此选项的情况下启用广播，Redis 将为每个 key
  发送通知。您不能删除单个前缀，但可以通过禁用和重新启用跟踪来删除所有前缀。使用此选项会增加 O(N<sup>2</sup>) 的额外时间复杂度，其中 N 是跟踪的前缀总数。
- `OPTIN` ：当广播未激活时，通常不跟踪只读命令中的 key，除非在 `CLIENT CACHING yes` 命令后立即调用它们。
- `OPTOUT` ：当广播未激活时，通常在只读命令中跟踪 key，除非在 `CLIENT CACHING no` 命令后立即调用它们。
- `NOLOOP` ：不要发送有关此链接本身修改的 key 的通知。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) : 如果连接成功进入跟踪模式或成功禁用跟踪模式，则 OK。否则返回错误。