## XPENDING key group [[IDLE min-idle-time] start end count [consumer]]

    起始版本：5.0.0.
    时间复杂度：O(N)，其中 N 是返回的元素数，因此每次调用要求固定的少量条目是 O(1)。O(M)，其中 M 是与 IDEL 过滤器一起使用时扫描的条目总数。当命令只返回摘要并且消费者列表很小时，它在 O(1) 时间内运行；否则，迭代每个消费者需要额外的 O(N) 时间。

通过消费者组从 Stream 中获取数据，而不是确认这些数据，具有创建 _待处理条目(pending entries)_ 的效果。这在 [XREADGROUP](xreadgroup.md) 命令中已有详尽的说明，在我们的 [introduction to Redis Streams](../topics/streams-intro.md) 中。[XACK](xack.md) 命令会立即从待处理条目列表 (pending entries list, PEL) 中移除大待处理条目，因为一旦消息被处理成功，消费者组就不再需要跟踪它并记住消息的当前所有者。

[XPENDING](xpending.md) 命令是检查待处理消息列表的接口，因此它是一个非常重要的命令，用于观察和了解消费者组正在发生的事情：哪些客户端是活跃的，哪些消息在等待消费者，或者查看是否有空闲的消息。此外，该命令与 [XCLAIM](xclaim.md) 一起使用，用于实现长时间故障的消费者的无法恢复，因此没有处理某些消息的情况：不同的消费者可以认领该消息并继续处理。这在 [stream intro](../topics/streams-intro.md) 和 [XCLAIM](xclaim.md) 命令页面中有更好地解释，这里不再介绍。

---

### Summary form of XPENDING

当只使用键名和消费者组名调用 [XPENDING](xpending.md) 时，其只输出有关给定消费者组的待处理消息的概要。在以下例子中，我们创建一个使用过的消费者组，并通过使用 [XREADGROUP](xreadgroup.md) 从组中读取来立即创建待处理消息：

```
> XGROUP CREATE mystream group55 0-0
OK

> XREADGROUP GROUP group55 consumer-123 COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) 1526984818136-0
         2) 1) "duration"
            2) "1532"
            3) "event-id"
            4) "5"
            5) "user-id"
            6) "7782813"
```

我们希望消费者组 group55 的待处理条目列表立即拥有一条消息：消费者 consumer-123 获取了一条消息，且没有确认消息。简单的 [XPENDING](xpending.md) 形式会给我们提供以下信息：

```
> XPENDING mystream group55
1) (integer) 1
2) 1526984818136-0
3) 1526984818136-0
4) 1) 1) "consumer-123"
      2) "1"
```

在这种形式中，此命令输出该消费者组的待处理消息的数量（即 1），然后是待处理消息的最小和最大 ID，然后列出消费者组中每一个至少一条待处理消息的消费者，以及它们的待处理消息数量。

---

### Extended form of XPENDING

这是一个很好地概述，但有时候我们对细节感兴趣。为了查看具有更多相关信息的所有待处理消息，我们还需要传递一系列 ID，与我们使用 [XRANGE](xrange.md) 时类似，以及一个非可选的 _count_ 参数，限制每一次调用返回的消息数量：

```
> XPENDING mystream group55 - + 10
1) 1) 1526984818136-0
   2) "consumer-123"
   3) (integer) 196415
   4) (integer) 1
```

在扩展的形式中，我们不再看到概要信息，而是在待处理消息列表中有每一条消息的详细信息。对于每条消息，返回四个属性：

1. 消息的 ID。
2. 获取并仍然要去人消息的消费者名称，我们称之为消息的当前 _所有者(owner)_ 。
3. 自上次将此消息传递给该消费者以来，经过的毫秒数。
4. 该消息被传递的次数。

交付计数器(deliveries counter)，即数组的第四个元素，当其他消费者使用 [XCLAIM](xclaim.md) *认领(claims)* 消息时，或当通过 [XREADGROUP](xreadgroup.md) 再次传递消息时，当访问消费者组中的消费者历史时（更多信息请参阅 [XREADGROUP](xreadgroup.md) 页面）递增。

最后，还可以向该命令传递一个额外的参数，以便查看具有特定所有者的消息：

```
> XPENDING mystream group55 - + 10 consumer-123
```

但是在上面的例子中，输出将是相同的，因为我们只有一个消费者有待处理消息。然而，我们需要记住的重要一点是，即使来自许多消费者的许多待处理消息，有特定消费者过滤的这种操作效率也不高：我们在全局和每个消费者都有待处理条目数据结构，所以我们可以非常高效地显示单个消费者的待处理消息。

---

### Idle time filter

从 6.2 版本开始，可以通过空闲时间过滤条目，以毫秒为单位（对于一段时间未处理的 XCLAIMing 条目有时很有用）：

```
> XPENDING mystream group55 IDLE 9000 - + 10
> XPENDING mystream group55 IDLE 9000 - + 10 consumer-123
```

第一种情况返回整个消费者组的前 10 个（或更少）空闲时间超过 9 秒的 PEL 条目，而在第二种情况仅返回 consumer-123 的满足条件的条目。

---

### Exclusive ranges and iterating the PEL

[XPENDING](xpending.md) 命令允许迭代挂起的条目，就像 [XRANGE](xrange.md) 和 [XREVRANGE](xrevrange.md) 允许 Stream 的条目一样。你可以通过在 `最后读取的(last-read)` 挂起条目的 ID 前面加上 `(` 字符，来表示开放（独占）范围区间，并在随便的命令调用中来证明它来实现这一点。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：

该命令以不同的格式返回数据，具体取决于它的调用方式，如本文前面所述。但是，返回值始终是一组 item。

---

### History

- &gt;= 6.2.0：增加 IDLE 选项和 exclusive 区间间隔。