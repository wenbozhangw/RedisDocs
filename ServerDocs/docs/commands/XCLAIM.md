## XCLAIM key group consumer min-idle-time ID [ID ...] [IDLE ms] [TIME ms-unix-time] [RETRYCOUNT count] [FORCE] [JUSTID]

    起始版本：5.0.0.
    时间复杂度：O(log N)，其中 N 是消费者组的 PEL 中的消息数。

在 Stream 的消费者组上下文中，此命令改变待处理消息的所有权，因此新的所有者是在命令参数中指定的消费者。通常是这样：
1. 假设有一个具有管理消费者组的 Stream。
2. 某个消费者 A 在消费者组的上下文中通过 [XREADGROUP](XREADGROUP.md) 从 Stream 中读取了一条消息。
3. 作为读取消息的副作用，消费者组的待处理条目列表(`Pending Entries List`, PEL)中创建了一个待处理消息条目：这意味着这条消息已经传递给给定的消费者，但是尚未通过 [XACK](XACK.md) 确认。
4. 突然这个消费者出现故障，且永远无法恢复。
5. 其他消费者可以使用 [XPENDING](XPENDING.md) 命令检查已经过时很长时间的待处理消息列表，为了继续处理这些消息，它们使用 [XCLAIM](XCLAIM.md) 来获得消息的所有权，并继续处理。

[Stream intro documentation](../topics/streams-intro.md) 中清楚的解释了这种动态过程。

请注意，消息只有在其空闲时间大于我们通过 [XCLAIM](XCLAIM.md) 指定的空闲时间时才会被认领。因为作为一个副作用，[XCLAIM](XCLAIM.md) 也会重置消息的空闲时间（因为这是处理消息的一次新尝试），两个试图同时认领消息的消费者永远不会成功：只有一个消费者能成功认领消息。这避免了我们用微不足道的方式多次处理给定的消息（虽然一般情况下无法完全避免多次处理）。

此外，作为副作用，[XCLAIM](XCLAIM.md) 会增加消息的尝试交付次数。通过这种方式，由于某些原因而无法处理的消息（例如，因为消费者在尝试处理期间崩溃），将会具有更大的计数器，并可以在系统内部被检测到。

---

### Command options

该命令有多个选项，但是大部分主要用于内部使用，以便将 [XCLAIM](XCLAIM.md) 或其他命令的结果传递到 AOF 文件，以及传递相同的结果到从节点，并且不太可能对普通用户有用：

1. `IDLE <ms>`：设置消息的空闲时间（自最后一次交付到目前的时间）。如果没有指定 IDLE，则假设 IDLE 值为 0，即时间计数被重置，因为消息现在有新的所有者来尝试处理它。
2. `TIME <ms-unix-time>`：这个命令与 IDLE 相同，但它不是设置相对的毫秒数，而是将空闲时间设置为一个指定的 Unix 时间（以毫秒为单位）。这对于重写生成 [XCLAIM](XCLAIM.md) 命令的 AOF 文件文有用。
3. `RETRYCOUNT <count>`：将重试计数器设置为指定的值。这个计数器在每次消息被交付的时候递增。通常，[XCLAIM](XCLAIM.md) 不会更改这个计数器，它只在调用 [XPENDING](XPENDING.md) 命令时提供给客户端：这样客户端可以检测到异常，例如，在大量传递尝试后，由于某种原因从未处理过的消息。
4. `FORCE`：在待处理条目列表(PEL)中创建待处理消息条目，即使某些指定的 ID 尚未在分配给不同客户端的待处理条目列表(PEL)中。但是消息必须存在于 Stream 中，否则不存在的消息 ID 将会被忽略。
5. `JUSTID`：只返回成功认领的消息 ID 数组，不返回实际的消息。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：

此命令以 [XRANGE](XRANGE.md) 相同的格式返回所有成功认领的消息。但如果指定了 `JUSTID` 选项，则只返回消息的 ID，不包括实际的消息。


---

### Examples

```
> XCLAIM mystream mygroup Alice 3600000 1526569498055-0
1) 1) 1526569498055-0
   2) 1) "message"
      2) "orange"
```

在上面的例子中，我们认领 ID 为 1526569498055-0 的消息，仅当消息闲置至少一个小时且没有原始消费者或其他消费者进行推进（确认或认领它）时，并将所有权分配给消费者 Alice。