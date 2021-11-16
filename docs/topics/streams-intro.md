## Introduction to Redis Streams

Stream 是 Redis 5.0 版本引入的一个新的数据类型，它以更抽象的方式模拟 *日志数据结构(log data structure)*，但日志仍然是完整的：就像一个日志文件，通常实现为以只附加模式打开的文件，Redis 流主要是一个仅附加数据结构。至少从概念来讲，因为 Redis 流是一种在内存表示的抽象数据类型，他们实现了更加强大的操作，以此来克服日志文件本身的限制。

---

### Streams basics

为了理解 Redis Stream 是什么以及如何使用他们，我们将忽略所有的高级特性，转而关注数据结构本身，即用于操作和访问它的命令。这基本上是大多数其他 Redis 数据类型共有的部分，比如 Lists、Sets、Sorted Sets 等等。然而，需要注意的是，Lists还有一个可选的、更加复杂的阻塞 API，由 **BLPOP** 等相似的命令导出。所以，从这方面来说，Streams 跟 Lists 并没有太大地不同，只是附加的 API 更复杂、更强大。

因为 Streams 是只附加(append only)数据结构，基本地写命令，叫 **XADD**，向指定的 Stream 追加一个新的条目。一个 Stream 条目不是简单的字符串，而是由一个或多个键值对组成。这样一来，Stream 的每一个条目就已经是结构化的，就像以 CSV 格式写的只附加文件一样，每一行由多个逗号隔开的字段组成。

```
> XADD mystream * sensor-id 1234 temperature 19.8
1518951480106-0
```

上面的例子中，调用的 **XADD** 命令往名称为 `mystream` 的 Stream 中添加了一个条目 `sensor-id: 1234, temperature: 19.8`，使用了自动生成的条目ID，也就是命令返回的值，也就是命令返回的值，具体在这里是 `1518951480106-0`。命令的第一个参数是 key 的名称 `mystream`，第二个参数是用于唯一确认 Stream 中每个条目的条目 ID。然而，在这个例子中，我们传入的参数值是 `*`，因为我们希望由 Redis 服务器为我们自动生成一个新的 ID。每一个新的 ID 都会单调增长，简单来讲就是，每次新添加的条目都会拥有一个比其他所有条目更大的 ID。有服务器自动生成 ID 与日志文件具有另一种相似性，即使用行号或者文件中的字节偏移量来识别一个给定的条目。回到我们的 **XADD** 例子中，跟在 key 和 ID 后面的参数是组成我们的 Stream 条目的键值对。

使用 **XLEN** 命令来获取一个 Stream 的条目数量。

```
> XLEN mystream
(integer) 1
```

---

#### Entry IDs

条目 ID 由 **XADD** 命令返回，并且可以唯一的标识给定 Stream 中的每一个条目，有两部分组成：

```
<millisecondsTime>-<sequenceNumber>
```

毫秒时间部分实际是生成 Stream ID 的 Redis 节点的服务器本地时间，但是如果当前毫秒时间戳比以前的条目时间戳小的话，那么会使用以前的条目时间，所以即便是服务器时钟向后跳，单调增长 ID 的特性仍然会保持不变。序列号用于以相同的毫秒创建的条目。由于序列号是 64 位的，所以实际上对于在同一毫秒内生成的条目数量是没有限制的。

这样的 ID 格式也许最初看起来有点奇怪，也许读者会好奇为什么时间会是 ID 的一部分。其实是因为 Redis Streams 支持按 ID 进行返回查询。由于 ID 与生成条目的时间相关，因此可以很容易地按时间范围进行查询。我们在后面讲到 **XRANGE** 命令时，很快就能明白这一点。

如果由于某些原因，用户需要与时间无关但实际上与另一个外部系统 ID 关联的增量 ID，就像前面所说的，**XADD** 命令可以带上一个显式的 ID，而不是使用通配符 `*` 来自动生成，如下所示：

```
> XADD somestream 0-1 field value
0-1
> XADD somestream 0-2 foo bar
0-2
```

请注意，在这种情况下，最小 ID 为 0-1，并且命令不接受等于或小于前一个 ID 的 ID ：

```
> XADD somestream 0-1 foo bar
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
```

---

### Getting data from Streams

现在我们终于能够通过 **XADD** 命令向我们的 Stream 中追加条目了。然而，虽然往 Stream 中追加数据非常明显，但是为了提取数据而查询 Stream 的方式并不是那么明显，如果我们继续使用日志文件进行类比，一种显而易见的方式是模拟我们通常使用 Unix 命令 `tail -f` 来做的事情，也就是，我们可以开始监听以获取追加到 Stream 的新消息。需要注意的是，不像 Redis 的阻塞列表，一个给定的元素只能到达某一个使用了 *pop style(冒泡风格)* 的阻塞客户端，比如使用类似 **BLPOP** 的命令，在 Streams 中我们希望看到的是多个消费者都能看到追加到 Stream 中的消息，就像许多的 `tail -f` 进程能同时看到追加到日志文件的内容一样。用传统属于来讲就是我们希望 Streams 可以 *fan out(扇形分发)* 消息到多个客户端。

然而，这只是其中一种可能的访问模式。我们还可以使用一种完全不同的方式来看待一个 Stream：不是作为一个消息传递系统，而是作为一个*time series store（时间序列存储）* 。在这种情况下，附加新消息可能也很有用，但是另一种自然查询模式是通过时间范围来获取消息，或者使用游标来增量遍历所有的历史消息。这绝对是另一种有用地访问模式。

最后，如果我们从消费者的角度来观察一个 Stream，我们也许想要以另外一种方式来访问它，那就是，作为一个可以分区到多个处理此类消息的多个消费者的消息流，以便消费者组只能看到到达单个流的消息的子集。

Redis Streams 通过不同的命令支持所有上面提到的三种访问模式。接下来的部分将展示所有这些模式，从最简单和更直接的使用：范围查询开始。

---

#### Querying by range: XRANGE and XREVRANGE

要根据范围查询 Stream，我们只需要提供两个 ID，即 _start_ 和 _end_ 。返回的区间的数据将会包括ID是 start 和 end 的元素，因此区间是完全包含的。两个特殊的 ID：`-` 和 `+` 分别表示可能的最小 ID 和最大 ID。

```
> XRANGE mystream - +
1) 1) 1518951480106-0
   2) 1) "sensor-id"
      2) "1234"
      3) "temperature"
      4) "19.8"
2) 1) 1518951482479-0
   2) 1) "sensor-id"
      2) "9999"
      3) "temperature"
      4) "18.2"
```

返回的每个条目都是有两个元素的数组：ID 和键值对列表。我们已经说过条目 ID 的与时间有关系，因为在字符 `-` 的左边的部分是创建 Stream 条目的本地节点上的 Unix 毫秒时间，即条目创建的那一刻（请注意：Streams 的复制使用的是 fully specified 的 **XADD** 命令，因此从节点将具有与主节点相同的 ID）。这意味着我可以使用 **XRANGE** 查询一个时间范围。然而为了做到这一点，我可能想要省略 ID 的序列号部分：如果省略，区间范围的开始序列号将默认为 0，结束部分的序列号默认是有效的最大序列号。这样一来，仅使用两个 Unix 毫秒时间去查询，我们就可以得到在那段时间内产生的所有条目（包含开始和结束）。例如，我可能想要查询两毫秒时间，可以这样使用：

```
> XRANGE mystream 1518951480106 1518951480107
1) 1) 1518951480106-0
   2) 1) "sensor-id"
      2) "1234"
      3) "temperature"
      4) "19.8"
```

我在这个范围内只有一个条目，然而在实际数据集中，我可以查询数小时的范围，或者两毫秒之间包含了许多的项目，返回的结果集很大。因此，**XRANGE** 命令支持在最后放一个可选的 **COUNT** 选项。通过指定一个 count，我可以只获取前面 N 个项目。如果我想要更多，我可以拿返回的最后一个 ID，在序列号部分加 1，然后再次查询。我们在下面的例子中看到这一点。我们开始使用 **XADD** 添加 10 个项目（我这里不具体展示，假设流 `mystream` 已经填充了10个项目）。要开始我的迭代，每个命令只获取 2 个项目，我从全范围开始，但 COUNT 是 2。

```
> XRANGE mystream - + COUNT 2
1) 1) 1519073278252-0
   2) 1) "foo"
      2) "value_1"
2) 1) 1519073279157-0
   2) 1) "foo"
      2) "value_2"
```

为了继续下面两个项目的迭代，我必须用返回最后一个ID，即 `1519073279157-0`，并在 ID 序列号部分加 1。请注意，序列号是 64 位的，因此无需检查溢出。在这个例子中，我们得到的结果 ID 是 `1519073279157-1`，现在可以用作下一次 **XRANGE** 调用的新的 start 参数：

```
> XRANGE mystream (1519073279157-0 + COUNT 2
1) 1) 1519073280281-0
   2) 1) "foo"
      2) "value_3"
2) 1) 1519073281432-0
   2) 1) "foo"
      2) "value_4"
```

依次类推。由于 **XRANGE** 的查找复杂度是 _O(log(N))_，因此返回 M 个元素为 _O(M)_，这个命令在 count 较小的时候，具有对数时间复杂度，这意味着每一步迭代速度都很快。所以 **XRANGE** 也是事实上的 _streams iterator(流迭代器)_，并且不需要 **XSCAN** 命令。

**XREVRANGE** 命令与 **XRANGE** 相同，但是以相反的顺序返回元素，因此 **XREVRANGE** 的实际用途是检查一个 Stream 中的最后一项是什么：

```
> XREVRANGE mystream + - COUNT 1
1) 1) 1519073287312-0
   2) 1) "foo"
      2) "value_10"
```

请注意：**XREVRANGE** 命令以相反的顺序获取 _start_ 和 _stop_ 参数。

---

### Listening for new items with XREAD

当我们不想按照 Stream 中的某个范围访问项目时，我们通常想要的是 _订阅_ 到达 Stream 的新条目。这个概念可能与 Redis 中 订阅频道的 Pub/Sub 或者Redis的阻塞队列有关，在这里等待某一个 key 去获取新的元素，但是这跟你消费 Stream 有这本质不同：

1. 一个 Stream 可以拥有多个客户端（消费者）在等待数据。默认情况下，对于每一个新项目，都会被分发到等待给定 Stream 的数据的每一个消费者。这个行为与阻塞队列不同的是，阻塞队列中每个消费者都会获取到不同的元素。但是，_fan out（扇形分发）_到多个消费者的能力与 Pub/Sub 相似。
2. 虽然在 Pub/Sub 中的消息是 _fire and forget_ 并且从不存储，以及使用阻塞列表时，当一个客户端收到消息时，它会从列表中弹出（有效删除），Stream 从根本上以一种不同的方式工作。所有的消息都被无限期地附加到 Stream 中（除非用户明确地要求删除这些条目）：不同的消费者通过记住收到的最后一条消息的 ID，从其角度知道什么是新消息。
3. Streams 消费者组提供了一种 Pub/Sub 或者阻塞列表都不能实现的控制级别，同一个 Stream 不同的群组，显式地确认已经处理的项目，检查待处理的项目的能力，申明未处理的消息，以及每个消费者拥有连贯历史可见性，单个客户端智能查看自己过去的消息历史记录。

提供监听到达 Stream 的新消息的能力的命令称为**XREAD**。比 **XRANGE** 要更复杂一点，所以我们将从简单的形式开始，稍后将提供整个命令布局。

```
> XREAD COUNT 2 STREAMS mystream 0
1) 1) "mystream"
   2) 1) 1) 1519073278252-0
         2) 1) "foo"
            2) "value_1"
      2) 1) 1519073279157-0
         2) 1) "foo"
            2) "value_2"
```

以上是 **XREAD** 的非阻塞形式。注意 **COUNT** 选项并不是必须的，实际上这个命令唯一强制的选项是 **STREAMS**，指定了一组 key 以及调用者已经看到的每个 Stream 响应的最大 ID，以便该命令仅向客户端提供 ID 大于我们指定 ID 的消息。

在上面的命令中，我们写了 `STREAMS mystream 0`，所以我们想要流 `mystream` 中所有 ID 大于 0-0 的消息。正如你在上面的例子中所看到的，命令返回了键名，因为时间上可以通过传入多个 key 来同时从不同的 Stream 中读取数据。我可以写一下，例如：`STREAMS mystream otherstream 0 0`。注意，在 **STREAMS** 选项后面，我们需要提供键名称，以及之后的 ID。因此，**STREAMS** 选项必须始终是最后一个。

除了 **XREAD** 可以同时访问多个 Stream，以及我们能够指定我们最后拥有的最后一个 ID 来获取之后的新消息，在一个简单的形式中，这个命令并没有做什么跟 **XRANGE** 有太大区别的事情。然而，有趣的部分是我们可以通过指定 **BLOCK** 参数，轻松地将 **XREAD** 编程一个 _blocking command(阻塞命令)_：

```
> XREAD BLOCK 0 STREAMS mystream $
```

请注意，在上面的例子中，除了移除 **COUNT** 以外，我指定了新的 **BLOCK** 选项，超时时间为 0 毫秒（意味着永不超时）。此外，我并没有给流 `mystream` 传入一个常规的 ID，而是传入了一个特殊的ID `$`。这个特殊的 ID 意思是 **XREAD** 应该使用流 `mystream` 已经存储的最大 ID 作为最后一个 ID。以便我们仅接收从我们开始监听时间以后的新消息。这在某种程度上类似于 Unix 命令 `tail -f`。

请注意当使用 **BLOCK** 选项时，我们不必使用特殊 ID `$`。我们可以使用任意有效的 ID。如果命令能够立即处理我们的请求而不会阻塞，它将执行此操作，否则它将阻止。通常如果我们想要从新的条目开始消费 Stream，我们以 `$` 开始，接着继续使用接收到的最后一条消息的 ID 来发起下一次请求，依次类推。

**XREAD** 的阻塞形式同样可以监听多个 Stream，只需要指定多个键名即可。如果请求可以同步提供，因为至少有一个流的元素大于我们指定的相应 ID，则返回结果。否则，该命令将阻塞并将返回获取新数据的第一个流的项目（根据提供的 ID）。

跟阻塞列表的操作类似，从等待数据的客户端角度来看，阻塞流读取是 _fair(公正的)_ ，由于语义是 FIFO 样式。阻塞给定 Stream 的第一个客户端是第一个在新项目可用时将被解除阻塞的客户端。

**XREAD** 命令并没有除了 **COUNT** 和 **BLOCK** 以外的其他选项，因此它是一个非常基本的命令，具有特定目的来攻击一个消费者或多个流。使用消费者组 API 可以用更强大的功能来消费 Stream，但是通过消费者组是通过另外一个不同的命令来实现的，成为 **XREADGROUP**。本指南的下一节将介绍。

---

### Consumer groups

当手头的任务是从不同客户端消费同一个 Stream，那么 **XREAD** 已经提供了一种方式可以 _fan out（扇形分发）_ 到 N 个客户端，还可以使用从节点来提供更多的读取可伸缩性。然而，在某些问题中，我们想要做的不是想许多客户端提供相同的消息流，而是从同一流向许多客户端提供不同的消息子集。一个很有用且很明显的例子是处理消息的速度很慢：能够让 N 个不同的客户端接收流的不同部分，通过将不同的消息路由到准备做更多工作的不同客户端来扩展消息处理工作。

实际上，假如我们想象有三个消费者C1，C2，C3，以及一个包含了消息1, 2, 3, 4, 5, 6, 7的Stream，我们想要按如下图表的方式处理消息：

```
1 -> C1
2 -> C2
3 -> C3
4 -> C1
5 -> C2
6 -> C3
7 -> C1
```

为了获得这个效果，Redis 使用了一个名为 _Consumer group (消费者组)_ 的概念。非常重要的一点是，从实现的角度来看，Redis 的消费者组与 Kafka(TM) 消费者组没有任何关系，它们只是从实施的概念上来看比较相似，所以我决定不改变最初普及这种想法的软件产品已有的术语。

消费者组就像一个 _pseudo consumer(伪消费者)_ ，从流中获取数据，实际上为多个消费者提供服务，提供某些保证：

1. 每条消息都提供给不同的消费者，因此不可能将相同的消息传递给多个消费者。
2. 消费者在消费者组中通过名称来识别，该名称是实施消费者的客户端必须选择的区分大小写的字符串。这意味着即便断开连接过后，消费者组仍然保留了所有的状态，因为客户端会重新申请成为相同的消费者。然而，这也意味着由客户端提供唯一的标识符。
3. 每一个消费者组都有一个 _first ID never consumed (第一个 ID 用于不被消费)_ 的概念，这样一来，当消费者请求新消息时，他能提供以前从未传递过的消息。
4. 消费消息需要使用特定的命令进行显式确认，表示：这条消息已经被正确处理了，所以可以从消费者组中逐出。
5. 消费者组跟踪所有当前待处理的消息，也就是，消息被传递到消费者组的一些消费者，但是还没有被确认为已经处理。由于这个特性，当访问一个 Stream 的历史消息的时候，每个消费者将 _only see messages that were delivered to it (只能看到传递给它的消息)_ 。

在某种程度上，消费者组可以被想象为关于Stream的 _amount of state(一些状态)_ ：

```
+----------------------------------------+
| consumer_group_name: mygroup           |
| consumer_group_stream: somekey         |
| last_delivered_id: 1292309234234-92    |
|                                        |
| consumers:                             |
|    "consumer-1" with pending messages  |
|       1292309234234-4                  |
|       1292309234232-8                  |
|    "consumer-42" with pending messages |
|       ... (and so forth)               |
+----------------------------------------+
```

如果你从这个视角来看，很容易理解一个消费者组能够做什么，如果做到向给定消费者提供它们的历史待处理消息，以及当消费者请求新消息的时候，是如何做到只发送 ID 大于 **last_delivered_id** 的消息的。同时，如果你把消费者组看成 Redis Stream 的辅助数据结构，很明显单个 Stream 可以拥有多个消费者组，每个消费者组都有一组消费者。实际上，同一个 Stream 甚至可以通过 **XREAD** 让客户端在没有消费者组的情况下读取，同时有客户端通过 **XREADGROUP** 在不同的消费者组中读取。

现在是时候放大来看基本的消费者组命令了，具体如下：

- **XGROUP** 用于创建，销毁或者管理消费者组。
- **XREADGROUP** 用于通过消费者组从一个 Stream 中读取。
- **XACK** 是允许消费者将待处理消息标记为已正确处理的命令。

---

#### Creating a consumer group

假设我已经存在 stream 类型的 `mystream`，为了创建消费者组，我只需要做：

```
> XGROUP CREATE mystream mygroup $
OK
```

请注意：目前还不能为不存在的 Stream 创建消费者组，但有可能在不就的将来我们会给 **XGROUP** 命令增加一个选项，以便在这种场景下可以创建一个空的 Stream。

如你所看到的上面这个命令，当创建一个消费者组的时候，我们必须指定一个ID，在这个例子中 ID 是 `$`。这是必要的，因为消费者组在其他状态中必须知道在第一个消费者连接时接下来要服务的消息，即消费者组创建完成时的 *最后消息ID (last message ID)* 是什么?如果我们就像上面例子一样，提供一个 `$`，那么只有从现在开始到达 Stream 的新消息才会被传递到消费者组中的消费者。如果我们指定的消息 ID 是 `0`，那么消费者组将会开始消费这个 Stream 中的 `所有` 历史消息。当然，你也可以指定任意其他有效的 ID。你所知道的是，消费者组将开始传递 ID 大于你所指定的 ID 的消息。因为 `$` 表示 Stream 中当前最大 ID 的意思，指定 `$` 会有只消费新消息的效果。

**XGROUP CREATE** 还支持自动创建流，如果它不存在，使用可选的 **MKSTREAM** 子命令作为最后一个参数：

```
> XGROUP CREATE newstream mygroup $ MKSTREAM
OK
```

现在消费者组创建好了，我们可以使用 **XREADGROUP** 命令立即开始尝试通过消费者组读取消息。我们会从消费者哪里读到，假设指定消费者分别是 Alice 和 Bob，来看看系统会怎么样返回不同消息给 Alice 和 Bob。

**XREADGROUP** 和 **XREAD** 非常类似，并且提供了相同的 **BLOCK** 选项，除此以外还是一个同步命令。但是有一个 _强制的(mandatory)_ 选项必须指定，那就是 **GROUP**，并且有两个参数：消费者组的名字，以及尝试读取的消费者的名字。选项 **COUNT** 仍然是支持的，并且与 **XREAD** 命令中的用法相同。

在开始从 Stream 中读取之前，让我们往里面放一些消息：

```
> XADD mystream * message apple
1526569495631-0
> XADD mystream * message orange
1526569498055-0
> XADD mystream * message strawberry
1526569506935-0
> XADD mystream * message apricot
1526569535168-0
> XADD mystream * message banana
1526569544280-0
```

请注意：_在这里消息是字段名称，水果是关联的值，记住 Stream 中的每一项都是小字典_ 。现在是时候尝试使用消费者组读取了：

```
> XREADGROUP GROUP mygroup Alice COUNT 1 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) 1526569495631-0
         2) 1) "message"
            2) "apple"
```

**XREADGROUP** 的响应内容就像 **XREAD** 一样。但是请注意上面提供的 `GROUP <group-name> <consumer name>`，这表示我想要使用消费者组 `mygroup` 从 Stream 中读取，我是消费者 Alice。每次消费者使用消费者组中执行操作时，都必须要指定可以在这个消费者组中唯一标识它的名字。

在以上命令行中还有另外一个非常重要的细节，在强制选项 **STREAMS** 之后，键 `mystream` 请求的 ID 是特殊的 ID `>`。这个特殊的 ID 只在消费者组的上下文中有效，其意思是：**到目前为止从未传递给其他消费者的消息**。

这几乎总是你想要的，但是也可以指定一个真实的 ID，比如 0 或者任何其他有效的 ID，在这个例子中，我们请求 **XREADGROUP** 只提供给我们 **历史待处理的消息(history of pending messages)**，在这种情况下，将永远不会在组中看到新消息。所以基本上 **XREADGROUP** 可以根据我们提供的 ID 有以下行为：
- 如果 ID 是指定的 ID `>`，那么命令将会返回到目前为止从未传递给其他消费者的消息，这有一个副作用，就是会更新消费者组的 *最后ID(last ID)* 。
- 如果 ID 是任意其他有效的数字 ID，那么命令将会让我们访问我们的 *历史待处理消息(history of pending messages)* 。即传递给这个指定消费者（由提供的名称标识）的消息集，并且到目前为止从未使用 **XACK** 进行确认。

我们可以立即测试此行为，指定 ID 为 0，不带任何 **COUNT** 选项：我们只会看到唯一的待处理消息，即关于 apples 的消息：

```
> XREADGROUP GROUP mygroup Alice STREAMS mystream 0
1) 1) "mystream"
   2) 1) 1) 1526569495631-0
         2) 1) "message"
            2) "apple"
```

但是，如果我们确认这个消息已处理，它将不再是历史待处理消息的一部分，因此系统将不再报告任何消息：

```
> XACK mystream mygroup 1526569495631-0
(integer) 1
> XREADGROUP GROUP mygroup Alice STREAMS mystream 0
1) 1) "mystream"
   2) (empty list or set)
```

如果你还不清楚 **XACK** 是如何工作的，请不用担心，这个概念是：已处理的消息不再是我们可以访问的历史记录的一部分。

现在轮到 Bob 来读取一些东西了：

```
> XREADGROUP GROUP mygroup Bob COUNT 2 STREAMS mystream >
1) 1) "mystream"
   2) 1) 1) 1526569498055-0
         2) 1) "message"
            2) "orange"
      2) 1) 1526569506935-0
         2) 1) "message"
            2) "strawberry"
```

Bob 要求最多两条消息，并通过统一消费者组 `mygroup` 读取。所以发生的是 Redis 仅报告 _新_ 消息。正如你所看到的，"message apple" 未被传递，因为他已经被传递给 Alice，所以 Bob 获取到了 orange 和 strawberry，以此类推。

这样，Alice、Bob 以及这个消费者组中的任何其他消费者，都可以从相同的 Stream 中读取到不同的消息，读取他们尚未处理的历史消息，或者标记消息为已处理。这允许创建不同的 _拓扑(topologies)_ 和 _语义(semantics)_ 来从 Stream 中消费消息。

有几件事需要记住：
- 消费者是在他们第一次被提及的时候自动创建的，不需要显示创建。
- 即使使用 **XREADGROUP**，你也可以同时从多个 key 中读取，但是要让其工作，你需要给每一个 Stream 创建一个名称相同的消费者组。这并不是一个常见的需求，但是需要说明的是，这个功能在技术上是可以实现的。
- **XREADGROUP** 命令是一个 _写命令(write command)_，因为当它从 Stream 中读取消息时，消费者组被修改了，所以这个命令只能在 master 节点调用。

使用 Ruby 语言编写的消费者组实现示例如下。Ruby代码的编写方式，几乎对使用任何其他语言编程的程序员或者不懂Ruby的人来说，都是清晰可读的：

```ruby
require 'redis'

if ARGV.length == 0
    puts "Please specify a consumer name"
    exit 1
end

ConsumerName = ARGV[0]
GroupName = "mygroup"
r = Redis.new

def process_message(id,msg)
    puts "[#{ConsumerName}] #{id} = #{msg.inspect}"
end

$lastid = '0-0'

puts "Consumer #{ConsumerName} starting..."
check_backlog = true
while true
    # Pick the ID based on the iteration: the first time we want to
    # read our pending messages, in case we crashed and are recovering.
    # Once we consumed our history, we can start getting new messages.
    if check_backlog
        myid = $lastid
    else
        myid = '>'
    end

    items = r.xreadgroup('GROUP',GroupName,ConsumerName,'BLOCK','2000','COUNT','10','STREAMS',:my_stream_key,myid)

    if items == nil
        puts "Timeout!"
        next
    end

    # If we receive an empty reply, it means we were consuming our history
    # and that the history is now empty. Let's start to consume new messages.
    check_backlog = false if items[0][1].length == 0

    items[0][1].each{|i|
        id,fields = i

        # Process the message
        process_message(id,fields)

        # Acknowledge the message as processed
        r.xack(:my_stream_key,GroupName,id)

        $lastid = id
    }
end
```

正如你所看到的，这里的想法是开始消费历史消息，即我们的待处理消息列表。这很有用，因为消费者可能已经崩溃，因此在重新启动时，我们想要重新读取那些已经传递给我们但还没有确认的消息。通过这种方式，我们可以多次或者一次处理消息（至少在消费者失败的场景中是这样，但是这也收到 Redis 持久化和复制的限制，请参阅有关此主题的特定部分）。

消费历史消息后，我们将得到一个空的消息列表，我们可以切换到 `>`，使用特殊 ID 来消费新消息。

---

### Recovering from permanent failures

上面的例子允许我们编写多个消费者参与同一个消费者组，每个消费者获取消息的一个子集进行处理，并且在故障恢复时重新读取各自的待处理消息。然而在现实世界中，消费者有可能永久地失败并且永远无法恢复。由于任何原因停止后，消费者的待处理消息会发生什么呢？

Redis 的消费者组提供了一个专门针对这种场景的特性，用以 _认领(claim)_ 给定消费者的待处理消息，这样一来，这些消息就会改变它们的所有者，并且被重新分配给其他消费者。这个特性是非常明确的，消费者必须检查待处理消息列表，并且必须使用特殊命令来认领特定的消息，否则服务器将把待处理的消息永久分配给旧消费者，这样不同的应用程序就可以选择是否使用这样的特性，以及使用它的方式。

这个过程的第一步是使用一个叫做 **XPENDING** 的命令，这个命令提供消费者组中待处理条目的可观察性。这是一个只读命令，他总是可以安全地调用，不会改变任何消息的所有者。在最简单的形式中，调用这个命令只需要两个参数，即 Stream 的名称和消费者组的名称。

```
> XPENDING mystream mygroup
1) (integer) 2
2) 1526569498055-0
3) 1526569506935-0
4) 1) 1) "Bob"
      2) "2"
```

当以这种方式调用的时候，命令只会输出给定消费者组的待处理消息总数（在本例中是两条消息），所有待处理消息中的最小和最大 ID，最后是消费者列表和每个消费者的待处理消息数量。我们只有 Bob 有两条待处理消息，因为 Alice 七扭去的唯一一条消息已使用 **XACK** 确认了。

我们可以通过给 **XPENDING** 命令传递更多的参数来获取更多信息，完整的命令签名如下：

```
XPENDING <key> <groupname> [<start-id> <end-id> <count> [<consumer-name>]]
```

通过提供一个开始和结束 ID（可以只是 - 和 +，就像 **XRANGE** 一样），以及一个控制命令返回的信息量的数字，我们可以了解有关待处理消息的更多信息。如果我们想要将输出限制为仅针对给定消费者的待处理消息，可以使用最后一个可选参数，即消费者的名称，但我们不会在以下示例中使用此功能。

```
> XPENDING mystream mygroup - + 10
1) 1) 1526569498055-0
   2) "Bob"
   3) (integer) 74170458
   4) (integer) 1
2) 1) 1526569506935-0
   2) "Bob"
   3) (integer) 74170458
   4) (integer) 1
```

限制我们有了每一条消息的详细信息：消息ID，消费者名称，_空闲时间(idle time)_（单位是毫秒，意思是：自上次将消息传递给某个消费者以来经过了多少毫秒），以及每一条给定的消息被传递了多少次。我们有来自 Bob 的两条消息，它们空闲了 74170458 毫秒，大概 20 个小时。

请注意，没有人阻止我们检查第一条消息内容是什么，使用 **XRANGE** 即可。

```
> XRANGE mystream 1526569498055-0 1526569498055-0
1) 1) 1526569498055-0
   2) 1) "message"
      2) "orange"
```

我们只需要在参数中重复两次相同的 ID。现在我们有了一些想法，Alice 可能会根据 20 个小时仍没有处理这些消息，来判断 Bob 可能无法及时恢复，所以现在是时候 _认领(claim)_ 这些消息，并继续代替 Bob 处理了。为了做到这一点，我们使用 **XCLAIM** 命令。

这个命令非常地复杂，并且在其完整形式中有很多选项，因为它用于复制消费者组的更改，但我们只使用我们通常需要的参数。在这种情况下，他就想调用它一样简单：

```
XCLAIM <key> <group> <consumer> <min-idle-time> <ID-1> <ID-2> ... <ID-N>
```

基本上我们说，对于这个特定的 Stream 和消费者组，我们希望指定的 ID 的这些消息可以改变他们的所有者，并将被分配到指定的消费者 `<consumer>`。但是，我们还提供了最小空闲时间，因此只有在上述消息的空闲时间大于指定的空闲时间时，操作才会起作用。这很有用，因为有两个客户端会同时尝试认领一条消息：

```
Client 1: XCLAIM mystream mygroup Alice 3600000 1526569498055-0
Client 2: XCLAIM mystream mygroup Lora 3600000 1526569498055-0
```

然而认领一条消息的副作用是会重置它的空闲时间！并将增加其传递次数的计数器，所以上面第二个客户端的认领会失败。通过这种方式，我们可以避免堆消息进行简单的重新处理（即使是在一般情况下，你仍然不能获得准确的一次处理）。

下面是命令执行的结果：

```
> XCLAIM mystream mygroup Alice 3600000 1526569498055-0
1) 1) 1526569498055-0
   2) 1) "message"
      2) "orange"
```

Alice 成功认领了该消息，现在可以处理并确认消息，尽管原来的消费者还没有恢复，也能继续进行处理。

从上面的例子能很明显地看到，作为成功认领了指定消息的副作用，**XCLAIM** 命令也返回了消息数据本身。但这不是强制性的。可以使用 **JUSTID** 选项，已便金返回成功认领的消息的ID。如果你想减少客户端和服务器之间的带宽使用量的话，以及考虑命令的性能，这会很有用，并且你不会对消息感兴趣，因为稍后你的消费者的实现方式将不时地重新扫描历史待处理消息。

认领也可以通过一个独立的进程来实现：这个进程只负责检查待处理消息列表，并将空闲的消息重新分配给看似活跃的消费者。可以通过 Redis Stream 的可观察特性获得活跃的消费者。这是下一个章节的主题。

---

### Automatic claiming

Redis 6.2 中添加了 [XAUTOCLAIM](../commands/xautoclaim.md) 命令实现了我们上面描述的声明过程。

[XPENDING](../commands/xpending.md) 和 [XCLAIM](../commands/xclaim.md) 为不同类型的恢复机制提供了基本地构建块。此命令通过让 Redis 管理它来优化通用过程，并为大多数恢复需求提供简单的解决方案。

[XAUTOCLAIM](../commands/xautoclaim.md) 识别空闲的待处理消息并将它们的所有权转移给消费者。命令的签名如下所示：

```
XAUTOCLAIM <key> <group> <consumer> <min-idle-time> <start> [COUNT count] [JUSTID]
```

因此，在上面的示例中，我可以使用自动认领来声明一条消息，如下所示：

```
> XAUTOCLAIM mystream mygroup Alice 3600000 0-0 COUNT 1
1) 1526569498055-0
2) 1) 1526569498055-0
   2) 1) "message"
      2) "orange"
```

与 [XCLAIM](../commands/xclaim.md) 一样，该命令用声明的消息数组进行回复，但它也返回一个 Stream ID，允许迭代待处理的条目。Stream ID 是一个游标，我可以在下一次调用中使用它来继续认领空闲挂起的消息：

```
> XAUTOCLAIM mystream mygroup Lora 3600000 1526569498055-0 COUNT 1
1) 0-0
2) 1) 1526569506935-0
   2) 1) "message"
      2) "strawberry"
```

当 [XAUTOCLAIM](../commands/xautoclaim.md) 返回 "0-0" 流 ID 作为游标时，这意味着它到达了消费者组待处理条目列表的末尾。这并不意味着没有新的空闲挂起消息，因此该过程通过从流的开通调用 [XAUTOCLAIM](../commands/xautoclaim.md) 来继续。

---

### Claiming and the delivery counter

在 **XPENDING** 的输出中，你所看到的计数器是每一条消息的交付次数。这样的计数器以两种方式递增：消息通过 **XCLAIM** 成功认领时，或者调用 **XREADGROUP** 访问历史待处理消息时。

当出现故障时，消息被多次传递是很正常的，但最终它们通常会得到处理。但有时候处理特定的消息会出现问题，因为消息会以触发处理代码中的 bug 的方式被损坏或修改。在这种情况下，消费者处理这条特殊的消息会一直失败。因为我们有传递尝试的计数器，所以我们可以使用这个计数器来检测由于某些原因根本无法处理的消息。所以一旦消息的传递计算器达到你给定的值，比较明智的做法是将这些消息放入另外一个 Stream，并给系统管理员发送一条通知。这基本上是 Redis Stream 实现的 _dead letter_ 概念的方式。

---

### Streams observability

缺乏可观察性的消息系统很难处理。不知道谁在消费消息，哪些消息待处理。不知道给定 Stream 的活跃消费者组的集合，使得一切都不透明。因此，Redis Stream 和消费者组都有不同的方式来观察正在发生的事情。我们已经介绍了 **XPENDING**，它允许我们价差在给定时刻正在处理的消息列表，以及它们的空闲时间和传递次数。

但是，我们可能希望做更多的事情，**XINFO** 命令是一个可观察性接口，可以与子命令一起使用，以获取有关 Stream 或消费者组的信息。

这个命令使用子命令来显示有关 Stream 和消费者组的状态的不同信息，比如使用 **XINFO STREAM** 可以报告关于 Stream 本身的信息。

```
> XINFO STREAM mystream
 1) length
 2) (integer) 13
 3) radix-tree-keys
 4) (integer) 1
 5) radix-tree-nodes
 6) (integer) 2
 7) groups
 8) (integer) 2
 9) first-entry
10) 1) 1526569495631-0
    2) 1) "message"
       2) "apple"
11) last-entry
12) 1) 1526569544280-0
    2) 1) "message"
       2) "banana"
```

输出显示了有关如何在内部编码 Stream 的信息，以及显示了 Stream 的第一条和最后一条消息。另一个可用的信息是与这个 Stream 相关联的消费者组的数量。我们可以进一步挖掘有关消费者组的更多信息。

```
> XINFO GROUPS mystream
1) 1) name
   2) "mygroup"
   3) consumers
   4) (integer) 2
   5) pending
   6) (integer) 2
   7) last-delivered-id
   8) "1588152489012-0"
2) 1) name
   2) "some-other-group"
   3) consumers
   4) (integer) 1
   5) pending
   6) (integer) 0
   7) last-delivered-id
   8) "1588152498034-0"
```

正如你在这里和前面的输出中看到的，**XINFO** 命令输出一系列键值对。因为这是一个可观察性命令，允许人类用户立即了解报告的信息，并允许命令通过添加更多字段来报告更多信息，而不会破坏与旧客户端的兼容性。其他更高带宽肖略的命令，比如 **XPENDING**，只报告没有字段名称的信息。

上面例子中的输出（使用了子命令 **GROUPS**） 应该能够清楚地观察字段名称。我们可以通过检查在此类消费者组中注册的消费者，来更详细地检查特定消费者组的状态。

```
> XINFO CONSUMERS mystream mygroup
1) 1) name
   2) "Alice"
   3) pending
   4) (integer) 1
   5) idle
   6) (integer) 9104628
2) 1) name
   2) "Bob"
   3) pending
   4) (integer) 1
   5) idle
   6) (integer) 83841983
```

如果你不记得命令的语法，只需要查看命令本身的帮助：

```
> XINFO HELP
1) XINFO <subcommand> arg arg ... arg. Subcommands are:
2) CONSUMERS <key> <groupname>  -- Show consumer groups of group <groupname>.
3) GROUPS <key>                 -- Show the stream consumer groups.
4) STREAM <key>                 -- Show information about the stream.
5) HELP                         -- Print this help.
```

---

### Differences with Kafka(TM) partitions

Redis Stream 的消费者组可能类似于基于 Kafka(TM) 分区的消费者组，但是要注意 Redis Stream 实际上非常不同。分区仅仅是 _逻辑(logical)_ 的，并且消息只是放在一个 Redis 键中，因此不同客户端的服务方式取决于谁准备处理新消息，而不是从那个分区客户端读取。例如，如果消费者 C3 在某一点永久故障，Redis 会继续服务 C1 和 C2，将新消息送达，就像现在只有两个 _逻辑(logical)_ 分区一样。

类似的，如果一个给定的消费者在处理消息方面比其他消费者快很多，那么这个消费者在相同单位时间内按比例会接收更多的消息。这是有可能的，因为 Redis 显示地追踪所有未确认的消息，并且记住了谁接收了哪些消息，以及第一条消息的 ID 从未传递给任何消费者。

但是，这也意味着在 Redis 中，如果你真的想把同一个 Stream 的消息分区到不同的 Redis 实例中，你必须使用多个 key 和一些分区系统，比如 Redis Cluster 或者特定应用程序的分区系统。单个 Redis Stream 不会自动分区到多个实例上。

我们可以说，以下是正确的：
- 如果你使用一个 Stream -> 1 consumers，则消息是按顺序处理的。
- 如果你使用 N Stream -> N consumers，那么只有给定的消费者命中 N 个 Stream 的子集，你可以扩展上面的模型来实现 1 Stream -> 1 consumer。
- 如果你使用 1 Stream -> N consumers，则对 N 个消费者进行负载均衡，但是在那种情况下，有关同一逻辑项的消息可能会无序消耗，因为给定的消费者处理消息 3 可能比另一个消费者处理消息 4 要快。

所以基本上 Kafka 分区更现实使用了 N 个不同的 Redis 键。而 Redis 消费者组是一个将给定 Stream 的消息负载均衡到 N 个不同消费者的服务器负载均衡系统。

---

### Capped Streams

许多应用并不希望数据永久收集到一个 Stream。有时在 Stream 中指定一个最大项目数很有用，之后一旦达到给定的大小，将数据从 Redis 中移到不那么快的非内存存储是很有用的，适合用来记录未来几十年的历史数据。Redis Stream 对此有一定地支持。这就是 **XADD** 命令的 **MAXLEN** 选项，这个选项用起来很简单：

```
> XADD mystream MAXLEN 2 * value 1
1526654998691-0
> XADD mystream MAXLEN 2 * value 2
1526654999635-0
> XADD mystream MAXLEN 2 * value 3
1526655000369-0
> XLEN mystream
(integer) 2
> XRANGE mystream - +
1) 1) 1526654999635-0
   2) 1) "value"
      2) "2"
2) 1) 1526655000369-0
   2) 1) "value"
      2) "3"
```

如果使用 **MAXLEN** 选项，当 Stream 达到指定长度后，旧的条目会自动被逐出，因此 Stream 的大小是恒定的。目前还没有选项让 Stream 只保留给定数量的条目，因为为了一致地运行，这样的命令必须为了逐出条目而潜在地阻塞很长时间。比如可以想象一下如果存在插入尖峰，然后是长暂停，以及另一次插入，全都具有相同的最大时间。Stream 会阻塞来逐出在暂停期间变得太旧的数据。因此，用户需要进行一些规划，并了解 Stream 所需的最大长度。此外，虽然 Stream 的长度与内存使用是成正比的，但是按时间来缩减不太容易控制和预测：这取决于插入速率，改变了通常随时间变化（当它不变化时，那么按尺寸缩减是微不足道的）。

然而使用 **MAXLEN** 进行修整可能很昂贵：Stream 由宏节点表示为基数树，以便节省内存。改变由几十个元素组成的单个宏节点不是最佳的。因此可以使用以下特殊形式提供命令：

```
XADD mystream MAXLEN ~ 1000 * ... entry fields here ...
```

在选项 **MAXLEN** 和实际计数中间的参数 `~` 的意思是，我不是真的需要精确 1000 个项目。他可以是 1000 或者 1010 或者 1030，只要保证至少保存 1000 个项目就行。通过使用这个参数，仅当我们移除整个节点的时候才执行修正。这使得命令更高效，而且这也是我们通常想要的。

还有 **XTRIM** 命令可用，它做的事情与上面将到的 **MAXLEN** 选项非常相似，但是这个命令不需要添加任何其他参数，可以以独立的方式使用：

```
 XTRIM mystream MAXLEN 10
```

或者，对于 **XADD** 选项：

```
> XTRIM mystream MAXLEN ~ 10
```

但是，**XTRIM** 旨在接受不同的修整策略，虽然现在只实现了 **MAXLEN**。鉴于这是一个明确的命令，将来可能允许按时间来进行修修整，因为以独立的方式调用这个命令的用户应该知道他正在做什么。

一个有用地逐出策略是，**XTRIM** 应该具有通过一系列 ID 删除的能力。目前这是不可能的，但是将来可能会实现，以便更方便地使用 **XRANGE** 和 **XTRIM** 来将 Redis 中的数据迁移到其他存储系统中（如果需要）。

---

### Special IDs in the streams API

你可能已经注意到有几个特殊的 ID 可以在 Redis API 中使用。这里是一个简短的概述，以便它们在未来更有意义。

前两个特殊 ID 是 `-` 和 `+`，在 [XRANGE](../commands/xrange.md) 命令的范围查询中使用。这两个 ID 分布表示可能的最小 ID（基本上是 0-1 ）和可能的最大ID（基本上是 18446744073709551615-18446744073709551615）。正如你所看到的，用 `-` 和 `+` 代替这些数字要干净得多。

还有一些 API 我们想说，Stream 中 ID 最大项目的 ID。这就是 `$` 的含义。因此，例如，如果我只想是用 [XREADGROUP](../commands/xreadgroup.md)创建新条目，那么我使用这个 ID 来表示我已经又有所有现有条目，但没有将来要插入的新条目。类似的，当我创建或设置消费者组的 ID 时，我可以将最后交付的项设置为 `$`，以便指向改组中的消费者交付新条目。

正如你所看到的，`$` 并不意味着 `+`，它们是两个不同的东西，因为 `+` 是所有流的最大的 ID，而 `$` 是包含给定条目的给定流中的最大的 ID。此外，API 通常只理解 `+` 或 `$`，但它有助于避免加载具有多种含义的给定符号。

另一个特殊 ID 是 `>`，它的特殊含义仅与消费者组相关，且仅在使用 [XREADGROUP](../commands/xreadgroup.md) 命令时才具有。这个特殊的 ID 意味着我们只需要到目前为止从未交付给其他消费者的条目。 `>` ID 基本上是消费者组的最后一个传递 ID。

最后，只有XADD命令才能使用特殊的ID `*`，这意味着为我们的新条目自动选择一个ID。

所以我们有`-`，`+`，`$`，`>` 和 `*` ，它们都有不同的含义，大多数情况下，可以在不同的上下文中使用。

---

### Persistence, replication and message safety

与其他任何 Redis 数据结构一样，Stream 会异步复制到从节点，并持久化到 AOF 和 RDB 文件中。但可能不那么明显的是，消费者组的完整状态也会传输到 AOF，RDB 和从节点，因此如果消息在主节点是待处理的状态，在从节点也会是相同的信息。同样，节点重启后，AOF文件会恢复消费者组的状态。

但是请注意，Redis Stream 和消费者组使用 Redis 默认复制来进行持久化和复制，所以：
- 如果消息的持久性在您的应用程序中很重要，则 AOF 必须与强大的 fsync 策略一起使用。
- 默认情况下，异步复制不能保证复制 **XADD** 命令或者消费者组的状态更改：在故障转移后，可能会丢失某些内容，具体取决于从节点从主节点接收数据的能力。
- **WAIT** 命令可以用于强制将更改传输到一组从节点上。但是请注意，虽然这使得数据不太可能丢失，但由 Sentinel 或 Redis Cluster 运行的 Redis 故障转移过程禁止性 _尽力(effort)_ 检查以故障转移到最新的从节点，并且在某些特定故障下可能会选举出缺少一些数据的从节点。

因此，在使用 Redis Stream 和消费者组设计应用程序时，确保了解你的应用程序在故障期间应具有的语义属性，并进行响应地配置，评估它是否足够安全地用于您的用例。

---

### Removing single items from a stream

Stream 还有一个特殊的命令可以通过 ID 从中间移除项目。一般来讲，对于一个只附加的数据结构来说，这也许看起来是一个奇怪的特征，但实际上它对于设计例如隐私法规的应用程序是有用的。这个命令称为 **XDEL**，调用的时候只需要传递 Stream 的名称，在后面跟着需要删除的 ID 即可：

```
> XRANGE mystream - + COUNT 2
1) 1) 1526654999635-0
   2) 1) "value"
      2) "2"
2) 1) 1526655000369-0
   2) 1) "value"
      2) "3"
> XDEL mystream 1526654999635-0
(integer) 1
> XRANGE mystream - + COUNT 2
1) 1) 1526655000369-0
   2) 1) "value"
      2) "3"
```

但是在当前的实现中，在宏节点（macro node）完全为空之前，内存并没有真正回收，所以你不应该滥用这个特性。

---

### Zero length streams

Stream 与其他 Redis 数据结构有一个不同的地方在于，当其他数据结构没有元素的时候，调用删除元素的命令会把 key 本身删掉。举例来说，当调用 **ZREM** 命令将有序集合中的最后一个元素删除时，这个有序集合会被彻底删除。但 Stream 允许在没有元素的时候仍然存在，不管是因为使用 **MAXLEN** 选项的时候指定的 count 为零（在 **XADD** 和 **XTRIM** 命令中），或者因为调用了 **XDEL** 命令。

存在这种不对称性的原因是，Stream 可能具有相关联的消费者组，以及我们不希望因为 Stream 中没有项目而丢失消费者组定义的状态。当前，即使没有相关联的消费者组，Stream 也不会被删除，但这在将来可能会有变化。

---

### Total latency of consuming a message

非阻塞 Stream 命令，如 XRANGE 和 XREAD 或不带 BLOCK 选项的 XREADGROUP，与任何其他 Redis 命令一样被同步提供，所以讨论这些命令的延迟是没有意义的：检查 Redis 文件中命令的时间复杂度更有趣。可以这样说，在提取范围时，Stream 命令至少和 Sorted Set 命令一样快，而且 [XADD](../commands/xadd.md) 非常快，如果使用 pipeline，可以在平均机器中每秒插入 50万到 100万个项目。

但是，如果我们想要了解处理消息的延迟，在阻塞消费者组中的消费者的上下文中，从消息通过[XADD](../commands/xadd.md) 生成的那一刻到消费者获得消息的那一刻，延迟成为一个有趣的参数，因为 [XREADGROUP](../commands/xreadgroup.md) 随消息一起返回。

---

### How serving blocked consumers work

在提供执行测试的结果之前，了解 Redis 使用什么模型来路由 Stream 消息（以及实际上如何管理等待数据的任何阻塞操作）是很有趣的。

- 被阻塞的客户端在 hash table 中被引用，该哈希表将至少有一个阻塞消费者的 key 映射到等待此类key的消费者列表。这样，给定接收数据的 key，我们就可以解析所有正在等待此类数据的客户端。
- 当写入发生时，在这种情况下，当调用 [XADD](../commands/xadd.md) 命令时，它会调用 `signalKeyAsReady()` 函数。该函数会将 key 放入需要处理的 key 列表中，因为这些 key 可能对于被阻塞的消费者有新的数据。注意，这样的 _就绪键(ready keys)_ 会在后面处理，所以在同一个事件循环周期的过程中，键可能会收到其他写入。
- 最终，在返回事件循环之前，就绪键最终会被处理。对于每个 key，扫描等待数据的客户端列表，如果适用，这些客户端将接收到达的新数据。在 Stream 的情况下，数据是消费者请求的使用范围内的消息。

正如你所看到的，基本上，在返回时间循环之前，调用 [XADD](../commands/xadd.md) 的客户端和被阻塞消费消息的客户端都会在输出缓冲区中得到他们的回复，所以 [XADD](../commands/xadd.md) 的调用者应该得到回复，与此同时，消费者将获得新消息。

该模型是 *基于推送(push based)*，因为将数据添加到消费者缓冲区将通过调用 [XADD](../commands/xadd.md) 的操作直接执行，因此延迟往往是可以预测的。

---

### Latency tests results

为了检查这种延迟特性，我们使用多个 Ruby 程序实例执行了一个测试，这些实例将消息推送为计算机毫秒时间的附加字段，而 Ruby 程序则从消费者组读取消息并处理它们。消息处理步骤包括将当前计算机时间与消息时间戳进行比较，以了解总延迟。

这样的程序并没有优化，而是在一个运行Redis的两个核心的小实例中执行，以尝试提供在非最佳条件下可以预期的延迟数据。消息以每秒 10k 的速率生成，有 10 个消费者同时消费和确认来自同一 Redis Stream 和消费者组的消息。

获得的结果：

```
Processed between 0 and 1 ms -> 74.11%
Processed between 1 and 2 ms -> 25.80%
Processed between 2 and 3 ms -> 0.06%
Processed between 3 and 4 ms -> 0.01%
Processed between 4 and 5 ms -> 0.02%
```

因此 99.9% 的请求的延迟 <= 2 毫秒，异常值仍然非常接近平均值。

向 Stream 中添加几百万条未确认的消息不会改变 benchmark 的要点，大多数查询仍然以非常短的延迟处理。

几点说明：
- 这里我们每次迭代最多处理 10k 条消息，这意味着 XREADGROUP 的 COUNT 参数设置为 10000。这会增加很多延迟，但是为了让慢速消费者能够跟上消息流，这是必须的。因此，您可以预期真实世界的延迟要小得多。
- 与今天的标准相比，用于此 benchmark 的系统非常慢。