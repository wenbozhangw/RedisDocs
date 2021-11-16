## XREADGROUP GROUP group consumer [COUNT count] [BLOCK milliseconds] [NOACK] STREAMS key [key ...] ID [ID ...] 

    起始版本：5.0.0.
    时间复杂度：对于提到的每个流：O(M)，其中 M 是返回的元素数。如果 M 是常数（例如，总是用 COUNT 要求前 10 个元素），你可以认为它是 O(1)。另一方面，当 XREADGROUP 阻塞时，XADD 将使用 O(N) 时间，以便为在流中阻塞的 N 个客户端获取新数据提供服务。

[XREADGROUP](xreadgroup.md) 命令是 [XREAD](xread.md) 命令的特殊版本，支持消费者组。

在阅读本页之前，您可能必须了解 [XREAD](xread.md) 命令。

此外，如果您不熟悉流，我们建议您阅读我们 [introduction to Redis Streams](../topics/streams-intro.md) 。一定要理解介绍中消费者组的概念，这样下面这个命令的工作方式会更简单。

---

### Consumer groups in 30 seconds

此命令与普通 [XREAD](xread.md) 之间的区别在于，该命令支持消费者组。

如果没有消费者组，只需使用 [XREAD](xread.md)，所有客户端都将获得所有到达流的条目。

使用 [XREADGROUP](xreadgroup.md) 消费者组代替 [XREAD](xread.md)，可以创建客户端组，这些客户端使用到达给定流的消息的不同部分。例如，如果流获得新条目 A、B 和 C，并且有两个消费者通过消费者组读取，则一个客户端将获得消息 A 和 C，另一个客户端将获得消息 B，等等。

在消费者组中，给定的消费者（即，只是消费来自流的消息的客户端）必须使用唯一的*消费者名称*进行标识。这只是一个字符串。

消费者组的保证之一是给定的消费者只能看到传递给它的消息的历史记录，因此一条消息只有一个所有者。但是，有一个称为 _message claiming_ 的特殊功能，它允许其他使用者在某些使用者发生不可恢复的故障时认领消息。为了实现这样的语义，消费者组需要通过 [XACK](xack.md) 命令明确确认消费者成功处理的消息。这是必须的，因为流将针对每个消费者组跟踪谁在处理什么消息。

下面帮助您理解您是否需要消费者组：

1. 如果您有一个流和多个客户端，并且您希望所有客户端都获取所有消息，则不需要消费者组。
2. 如果您有一个流和多个客户端，并且您希望在您的客户端之间对流进行 _partitioned(分区)_ 或 _sharded(分片)_，以便每个客户端都将获得到达流中的消息的子集，则您需要一个消费者组。

---

### Differences between XREAD and XREADGROUP

从语法的角度来看，命令几乎是相同的，但是 [XREADGROUP](xreadgroup.md) _需要_ 一个特殊且强制的选项：

```
GROUP <group-name> <consumer-name>
```

组名只是与流关联的消费者组的名称。该组是使用 `XGROUP` 命令创建的。消费者名称是客户端用来在组内标识自己的字符串。第一次看到消费者时，它会在消费者组内自动创建。不同的客户端应该选择不同的消费者名称。

当您使用 [XREADGROUP](xreadgroup.md) 读取时，服务器将记住给定的消息已经发送给您：该消息将存储在消费者组中，称为 Pending Entries List(PEL)，即已发送但未确认的消息 ID 列表。

客户端必须使用 [XACK](xack.md) 确认消息处理，以便从 PEL 中删除挂起的条目。客户端使用 [XPENDING](xpending.md) 命令检查 PEL。

在不要求可靠性并且可以接受偶尔的消息丢失的情况下，可以使用 `NOACK` 子命令来避免将消息添加到 PEL。这相当于在消息被读取时确认消息。

使用 [XREADGROUP](xreadgroup.md) 时在 **STREAMS** 选项中指定的 ID 可以是以下两个之一：
- 特殊的ID `>`，这意味着消费者只想接收从未交付给任何其他消费者的消息。这只是意味着，给我新的消息。
- 任何其他 ID，即 0 或任何其他有效 ID 或不完整 ID（仅毫秒时间部分），将返回等待消费者发送 ID 大于提供的 ID 的命令的条目。所以基本上如果 ID 不是 `>`，那么该命令只会让客户端访问其挂起的条目：发送个给它的消息，但尚未确认。请注意，在这种情况下， `BLOCK` 和 `NOACK` 都被忽略。


与 [XREAD](xread.md) 一样，[XREADGROUP](xreadgroup.md) 命令可以以阻塞方式使用。在这方面没有任何区别。

---

### What happens when a message is delivered to a consumer ?

两件事情：

1. 如果消息从未发送给任何人，也就是说，如果我们正在谈论一条消息，则创建 PEL（Pending Entries List）。
2. 相反，如果消息已经传递给这个消费者，并且他只是再次重新获取相同的消息，那么最后一次传递计数器将更新为当前时间，并且传递次数加一。您可以使用 [XPENDING](xpending.md) 命令访问这些消息属性。

---

### Usage Example

通常您使用这样的命令来获取新消息并处理它们。在伪代码中：

```
WHILE true
    entries = XREADGROUP GROUP $GroupName $ConsumerName BLOCK 2000 COUNT 10 STREAMS mystream >
    if entries == nil
        puts "Timeout... try again"
        CONTINUE
    end

    FOREACH entries AS stream_entries
        FOREACH stream_entries as message
            process_message(message.id,message.fields)

            # ACK the message as processed
            XACK mystream $GroupName message.id
        END
    END
END
```

通过这种方式，示例消费者代码将仅获取新消息、处理它们、并通过 [XACK](xack.md) 确认它们。然而，上面的示例代码并不完整，因为他不处理崩溃后的恢复。如果我们在处理消息的过程中崩溃会发生什么，我们的消息将保留在待处理条目列表中，因此我们可以通过给 [XREADGROUP](xreadgroup.md) 最初的 ID 0 并执行相同的循环来访问我们的历史记录。一旦提供ID 0，UI回复是一组空消息，我们知道我们处理并确认了所有待处理的消息：我们可以开始使用 `>` 作为 ID，以便获得新消息并重新加入正在处理新消息的消费者。

要查看命令实际如何回复，请查看 [XREAD](xread.md) 命令页面。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：

该命令返回一个结果数组：返回数组的每个元素都是一个两个元素组成的数组，其中包含键名称和为该键报告的条目。报告的条目是完整的流条目，具有 ID 以及所有字段的值的列表。字段和值的报告顺序与 [XADD](xadd.md) 添加的顺序相同。

使用 **BLOCK** 时，超时时返回空回复。

强烈建议阅读 [Redis Streams introduction](../topics/streams-intro.md) ，以便获取更多地了解流的整体行为和语义。