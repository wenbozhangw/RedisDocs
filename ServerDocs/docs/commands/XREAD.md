## XREAD [COUNT count] [BLOCK milliseconds] STREAMS key [key ...] ID [ID ...]

    起始版本：5.0.0。
    时间复杂度：对于每个提到的流：O(N)，其中 N 是返回的元素数量，这意味着具有固定 COUNT 的 xread 是 O(1)。注意，当使用 BLOCK 选项时，XADD 将支付 O(M) 时间来为流上阻塞的 M 个客户端提供获取新数据的服务。

从一个或多个流读取数据，只返回 ID 大于调用者报告的最后接收 ID 的条目。此命令有一个阻塞选项，用户等待可用的项目，类似于 [BRPOP](BRPOP.md) 或 [BZPOPMIN](BZPOPMIN.md) 等等。

请注意，在阅读本页之前，如果你不了解 Stream，我们推荐先阅读 [our introduction to Redis Streams](/docs/topics/streams-intro.md) 。

---

### Non-blocking usage

如果未提供 **BLOCK** 选项，此命令是同步的，并可以认为与 [XRANGE](XRANGE.md) 有些相关：他将会返回流中的一系列项目，但与 [XRANGE](XRANGE.md) 相比它有两个基本差异（如果我们只考虑同步使用）：
- 如果我们想要从多个 key 同时读取，则可以使用多个流调用此命令。这是 [XREAD](XREAD.md) 的一个关键特性，因为特别是在使用 **BLOCK** 阻塞时，能够通过单个连接监听多个 key 是一个至关重要的特性。
- [XRANGE](XRANGE.md) 返回一组 ID 中的项目，[XREAD](XREAD.md) 更适合用于从第一个条目（比我们到目前为止看到的任何其他条目都要大）开始使用流。因此，我们传递给 [XREAD](XREAD.md) 的是，对于每个流，我们从该流接收的最后一个条目的ID。

例如，我又两个流 `mystream` 和 `writers`，并且我希望同时从这两个流中读取数据（从它们的第一个元素开始），我可以像下面这样调用 [XREAD](XREAD.md)。

请注意：我们在例子中使用了 **COUNT** 选项，因此对于每一个流，调用将返回每个流最多两个元素。

```
> XREAD COUNT 2 STREAMS mystream writers 0-0 0-0
1) 1) "mystream"
   2) 1) 1) 1526984818136-0
         2) 1) "duration"
            2) "1532"
            3) "event-id"
            4) "5"
            5) "user-id"
            6) "7782813"
      2) 1) 1526999352406-0
         2) 1) "duration"
            2) "812"
            3) "event-id"
            4) "9"
            5) "user-id"
            6) "388234"
2) 1) "writers"
   2) 1) 1) 1526985676425-0
         2) 1) "name"
            2) "Virginia"
            3) "surname"
            4) "Woolf"
      2) 1) 1526985685298-0
         2) 1) "name"
            2) "Jane"
            3) "surname"
            4) "Austen"
```

**STREAMS** 选项是强制的，并必须是最后一个项目，因为此选项按照下列格式获取可变长度的参数：

```
STREAMS key_1 key_2 key_3 ... key_N ID_1 ID_2 ID_3 ... ID_N
```

所以我们以一组流的 key 开始，并在后面跟着所有关联的 ID，表示我们*从该中获取的最后ID(the last ID we received for that stream)*，以便调用仅为我们提供同一流中具有更大 ID 的条目。

例如，在上面的例子中，我们从流 `mystream` 中接收到的最后项目的 ID 是 1526999352406-0，而对于流writers，我们接收的最后项目的ID是1526985685298-0。

要继续迭代这两个流，我将调用：

```
> XREAD COUNT 2 STREAMS mystream writers 1526999352406-0 1526985685298-0
1) 1) "mystream"
   2) 1) 1) 1526999626221-0
         2) 1) "duration"
            2) "911"
            3) "event-id"
            4) "7"
            5) "user-id"
            6) "9488232"
2) 1) "writers"
   2) 1) 1) 1526985691746-0
         2) 1) "name"
            2) "Toni"
            3) "surname"
            4) "Morris"
      2) 1) 1526985712947-0
         2) 1) "name"
            2) "Agatha"
            3) "surname"
            4) "Christie"
```

以此类推，最终，调用不再返回任何项目，只返回一个空数组，然后我们就知道我们的流中没有更多数据可以获取了（我们必须重试该操作，因此该命令也支持阻塞模式）。

---

### Incomplete IDs

使用不完整的 ID 是有效的，就像它对 [XRANGE](XRANGE.md) 一样有效。但是这里 ID 的序号部分，如果缺少，将总是被解释为 0，所以命令：

```
> XREAD COUNT 2 STREAMS mystream writers 0 0
```

完全等同于

```
> XREAD COUNT 2 STREAMS mystream writers 0-0 0-0
```

---

### Blocking for data

在同步形式中，只要有更多可用项，该命令就可以获取新数据。但是，有些时候，我们不得不等待数据生产者使用 [XADD](XADD.md) 向我们消费的流中推送新条目。为例避免使用固定或自适应区间获取数据，如果命令根据指定的流和 ID 不能返回任何数据，则该命令能够阻塞，并且一旦请求的 key 之一接收了数据，就会自动解除阻塞。

重要的是需要理解这个命令是 _扇形分发(fan out)_ 到所有正在等待相同 ID 范围的客户端，因此每个消费者都将得到一份数据副本，这与使用阻塞列表 pop 操作时发生的情况不同。

为了阻塞，可以使用 **BLOCK** 选项，以及我们希望在超时前阻塞的毫秒数。通常 Redis 阻塞命令的超时时间单位是秒，单词命令拥有一个毫秒超时时间，虽然通常服务器的超时时间精度大概在 0.1 秒左右。这可以在某些用例中阻塞更短的时间，并且如果服务器内部结构随着时间的推移而改善，它的超时精度也可能会有提升。

当传递了 **BLOCK** 选项，但是在传递的流中没有任何流有数据返回，那么该命令是同步执行的，就像没有*指定 BLOCK 选项一样(like if the BLOCK option would be missing)*。

这是阻塞调用的例子，其中命令稍后将返回空回复(nil)，因为超时时间已到，但是没用新的数据到达：

```
> XREAD BLOCK 1000 STREAMS mystream 1526999626221-0
(nil)
```

---

### The special $ ID

当使用阻塞时，有时我们希望只接收从我们阻塞的那一刻开始通过 [XADD](XADD.md) 添加到流中的条目。在这种情况下，我们队已经条件条目的历史不感兴趣。对于这个用例，我们不得不检查流的最大元素 ID，并使在 [XREAD](XREAD.md) 命令行中使用这个 ID。这不纯粹，并且需要调用其他命令，因此可以使用特殊的 ID `$` 来表名我们只想要流中的新条目。

你应该仅在第一次调用 [XREAD](XREAD.md) 时使用 `$` ，理解这一点**非常重要**。后面的 ID 你应该使用前一次报告的项目中的最后一项的 ID，否则你将会丢失在这期间，所有添加到这流中的条目。

这是典型的 [XREAD](XREAD.md) 调用（在想要仅消费新条目的消费者的第一次迭代）看起来的样子：

```
> XREAD BLOCK 5000 COUNT 100 STREAMS mystream $
```

一旦我们得到了一些回复，下一次调用会是像这样的：

```
> XREAD BLOCK 5000 COUNT 100 STREAMS mystream 1526999644174-3
```

依次类推。

---

### How multiple clients blocked on a single stream are served

列表或有序集合上的阻塞列表操作有 _pop_ 行为。基本上，元素将从列表或有序集合中删除，返回给客户端。在这种场景下，你希望 item 以公平的方式被消费，具体取决于客户端在给定 key 上阻塞的到达时刻。通常在这种用例中，Redis 使用 FIFO 语义。

但请注意，对于 Stream 这不是问题：当服务客户端时，不会从 Stream 中删除条目，因此只要 [XADD](XADD.md) 命令向流提供数据，就会提供给每个等待的客户端。

---

### Return Value

[Array reply](/docs/topics/protocol.md#resp-arrays)，特定的：

该命令返回一个数组结果：返回数组的每个元素都是一个由两个元素组成的数组（键名和为该键报告的条目）。报告的条目是完整的 Stream 条目，具有 ID 以及所有字段和值的列表。返回的条目及其字段和值的顺序与使用 [XADD](XADD.md) 添加它们的顺序完全一致。

当使用 **BLOCK** 时，超时时将返回一个空回复(nil)。

为了更多地了解流的整体行为和语义，强烈建议阅读 [Redis Stream introduction](/docs/topics/streams-intro.md)。

