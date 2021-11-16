## Pub/Sub

### Related commands

- [PSUBSCRIBE](../commands/psubscribe.md)
- [PUBLISH](../commands/publish.md)
- [PUBSUB CHANNELS](../commands/pubsub-channels.md)
- [PUBSUB HELP](../commands/pubsub-help.md)
- [PUBSUB NUMPAT](../commands/pubsub-numpat.md)
- [PUBSUB NUMSUB](../commands/pubsub-numsub.md)
- [PUNSUBSCRIBE](../commands/punsubscribe.md)
- [SUBSCRIBE](../commands/subscribe.md)
- [UNSUBSCRIBE](../commands/unsubscribe.md)

---

### Pub/Sub

[SUBSCRIBE](../commands/subscribe.md)、[UNSUBSCRIBE](../commands/unsubscribe.md) 和 [PUBLISH](../commands/publish.md) 实现了 发布/订阅消息传递范式 (Publish/subscribe messaging paradigm)，其中（引自维基百科）senders(publisher) 不是计划发送消息给特定的 receivers (subscribers)。相反，发布的消息被分到不同的频道，不需要知道什么样的订阅者订阅。订阅者对一个或多个频道感兴趣，只需接收感兴趣的消息，不需要知道什么样的发布者发布的。这种发布者和订阅者的解耦合可以带来更大的扩展性和更加动态的网络拓扑。

例如，为了订阅频道 foo 和 bar，客户端发出一个 [SUBSCRIBE](../commands/subscribe.md) 提供频道的名称：

```
SUBSCRIBE foo bar
```

其他客户端发送到这些频道的消息将被推送到所有订阅的客户端。


客户端订阅一个或多个频道不必发出命令，尽管它们订阅和取消订阅其他频道。 订阅和取消订阅操作的响应被封装在发送的消息中，以便客户端只需要读一个连续的消息流，其中第一个元素表示消息类型。订阅客户端上下文中允许的命令是 [SUBSCRIBE](../commands/subscribe.md)、[UNSUBSCRIBE](../commands/unsubscribe.md)、[PUNSUBSCRIBE](../commands/punsubscribe.md)、[PING](../commands/ping.md) 和 [QUIT](../commands/quit.md) 。

请注意，redis-cli 一旦进入订阅模式就不会接受任何命令，只能使用 Ctrl-C 退出该模式。

---

### Format of pushed messages

消息是一个包含三个元素的 [Array reply](../topics/protocol.md#resp-arrays)。

第一个元素是消息类型：

- `subscrube`：表示我们成功订阅到作为回复中第二个元素给出的频道。第三个参数代表我们当前订阅的频道数量。
- `unsubscribe`：表示我们成功取消订阅作为回复中第二个元素给出的频道。第三个参数代表我们当前订阅的频道数量。当最后一个参数为零时，我们不在订阅任何频道，客户端可以发出任何类型的 Redis 命令，因为我们处于 Pub/Sub 状态之外。
- `message`：这是另一个客户端 [PUBLISH](../commands/publish.md) 命令而收到的结果。第二个元素是来源频道的名称，第三个参数是实际消息的内容。

---

### Database & Scoping

Pub/Sub 与 keyspace 无关。他不会在任何级别上干扰他，包括数据库编号。

在 db 10 上发布，将被 db 1 上的订阅者听到。

如果您需要区分某些频道，可以通过在频道名称前面加上所在的环境的名称（例如：test, staging, production, ...）。

---

### Wire protocol example

```
SUBSCRIBE first second
*3
$9
subscribe
$5
first
:1
*3
$9
subscribe
$6
second
:2
```

此时，我们从另一个客户端对名为 second 的通道发出 [PUBLISH](../commands/publish.md) 操作：

```
> PUBLISH second Hello
```

这是第一个客户端收到的：

```
*3
$7
message
$6
second
$5
Hello
```

现在客户端使用不带附加参数的 [UNSUBSCRIBE](../commands/unsubscribe.md) 命令从所有频道取消订阅：

```
UNSUBSCRIBE
*3
$11
unsubscribe
$6
second
:1
*3
$11
unsubscribe
$5
first
:0
```

---

### Pattern-matching subscriptions

Redis Pub/Sub 实现支持模式匹配。客户端可以订阅 glob-style 模式以便接收所有来自能匹配到给定模式的频道的消息。

比如：

```
PSUBSCRIBE news.*
```

将接收所有发送到 `news.art.figurative、news.music.jazz ` 等等的消息，所有模式都是有效的，所以支持多通配符。

```
PUNSUBSCRIBE news.*
```

将取消订阅匹配该模式的客户端，这个调用不影响其他订阅。

由于模式匹配而收到的消息以不同的格式发送：

- 消息类型为 `pmessage`：它是作为另一个客户端发出的 [PUBLISH](../commands/publish.md) 命令的结果接收到的消息，匹配一个模式匹配订阅。第一个元素是元匹配的模式，第三个元素是原频道名称，最后一个元素是实际消息内容。

同样的，系统默认 [SUBSCRIBE](../commands/subscribe.md) 和 [UNSUBSCRIBE](../commands/unsubscribe.md)，[PSUBSCRIBE](../commands/psubscribe.md) 和 [PUNSUBSCRIBE](../commands/psubscribe.md) 命令在发送 `psubscribe` 和 `punsubscribe` 类型的消息时，使用像 `subscribe` 和 `unsubscribe` 一样的消息格式。

---

### Messages matching both a pattern and a channel subscription

客户端可能多次接收一个消息，如果它订阅的多个模式匹配了同一个发布的消息，或者他订阅的模式和频道同时匹配到一个消息。就像下面的例子：

```
SUBSCRIBE foo
PSUBSCRIBE f*
```

上面的例子中，如果一个消息被发送到foo,客户端会接收到两条消息：一条 `message` 类型，一条 `pmessage` 类型。

---

### The meaning of the subscription count with pattern matching

在 `subscribe`, `unsubscribe`, `psubscribe` 和 `punsubscribe` 消息类型中，最后一个参数是依旧活跃的订阅数量。这个数字是客户端依旧订阅的频道和模式的总数。只有当退订频道和模式的数量下降到 0 时，客户端才会退出 Pub/Sub 状态。

---

### Programming example

Pieter Noordhuis 提供了一个使用 EventMachine 和 Redis 创建 [a multi user high performance web chat](https://gist.github.com/pietern/348262) 。

---

### Client library implementation hints

因为所有接收到的消息包含导致消息传递的原始订阅（`message` 类型是频道，`pmessage` 类型是原始模式）客户端库可能会将原始订阅绑定到回到方法（可能是匿名函数，块，函数指针），使用 hash table。

当消息被接收的时候，可以做到时间复杂度为 O(1) 的查询，以便传递消息到已注册的回调。