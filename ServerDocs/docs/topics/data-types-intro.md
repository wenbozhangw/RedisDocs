## An introduction to Redis data types and abstractions

你也许已经知道 Redis 并不是简单的 key-value 存储，实际上它是一个 _data structures server_，支持不同类型的  值。也就是说，你不必仅仅把字符串当做 key 对应的值。下列这些数据类型都可以作为 value 的类型：
- Binary-safe strings.
- Lists：按插入顺序排序的字符串元素的集合。它们基本上就是 _链表（linked lists）_。
- Sets：不重复且无需的字符串元素的集合。
- Sorted Sets：类似 Sets，但是每个字符串元素都关联到一个叫 _score_ 浮点数值（floating number value）。里面的元素总是通过 score 进行排序，所以不同的是，它是可以检索的一系列元素。（例如你可能会问：给我前面10个或者后面10个元素）。
- Hashes：由 field 和关联的 value 组成的 map。field 和 value 都是字符串。这和 Ruby，Python 和 hashes 很像。
- Bit arrays (或者 simply bitmaps)：通过特殊的命令，你可以将 String 值当做一系列 bits 处理：可以设置和清除单独的 bits，统计所有设为 1 的 bits 的数量，找到最前的被设为 1 或 0 的 bit，等等。
- HyperLogLogs：这是被用于估计一个 set 中元素数量的概率性的数据结构。别害怕，它比看起来的样子要简单…参见本教程的 HyperLogLog 部分。
- Streams：提供抽象日志数据类型的类似 map entries 的 append-only 集合。[Introduction to Redis Streams](streams-intro.md) 对他们进行了深入介绍。 

学习这些数据类型的原理，以及如何使用它们解决 [command reference](https://redis.io/commands) 中的特定问题，并不总是那么容易。所以，本文档是一个关于 Redis 数据类型和他们最常见特性的导论。

在所有例子中，我们将使用 redis-cli 工具。它是一个简单而有用的命令行工具，用于向 Redis 服务器发出命令。

---

### Redis keys

Redis key 是二进制安全的，这意味着可以用任何二进制序列作为 key 值。从形如 "foo" 的简单字符串到一个 JPEG 文件的内容都可以。空字符串也是有效 key 值。

关于 key 的几条规则：
- 太长的 keys 不是个好主意，例如 1024 字节的 key 就不是个好主意，不仅因为消耗内存，而且在数据中查找这类 key 的计算成本也很高。
- 太短的 key 通常也不是个好主意，如果你要用 "u:1000:pwd" 来代替 "user:1000:password"，这没有什么问题，但后者更易阅读，并且由此增加的空间消耗相对于 key object 和 value object 本身来说很小。当然，没人阻止您一定要用更短的 key 节省一点空间。
- 最好坚持一种模式。例如："object-type:id:field" 就是个不错的主意，像这样 "user:1000:password"。我喜欢对多单词的字段名中间加上一个点，就像这样："comment:1234:reply.to" 或 "comment:1234:reply-to"。
- 允许的最大 key 大小为 512 MB。

---

### Redis Strings

Redis String 类型是您可以与 Redis key 关联的最简单的值类型。它是 Memcached 中唯一的数据类型，所以新手在 Redis 中使用它也是很自然的。

由于 Redis key 是字符串，当我们也是用字符串类型作为值时，我们将一个字符串映射到另一个字符串。字符串数据类型可用于许多用例，例如缓存 HTML 片段或页面。

让我们使用 redis-cli 来玩转字符串类型（在本教程中，所有实例都将通过 redis-cli 执行）。

```
> set mykey somevalue
OK
> get mykey
"somevalue"
```

正如您所见，使用 [SET](../commands/SET.md) 和 [GET](../commands/GET.md) 命令是我们设置和检索字符串值的方式。请注意，[SET](../commands/SET.md) 将替换已存储在 key 中的任何现有值，如果 key 已经存在，即使 key 与费字符串值相关，也会被替换。所以 [SET](../commands/SET.md) 执行一个赋值操作。

值可以是各种字符串（包括二进制数据），例如您可以在值中存储 jpeg 图像。值不能大于 512MB。

[SET](../commands/SET.md) 命令有一些有趣的选项，它们作为附加参数提供。例如，如果 key 已经存在，我可能会要求 [SET](../commands/SET.md) 失败，或者相反，只有当 key 已经存在时它才会成功：

```
> set mykey newval nx
(nil)
> set mykey newval xx
OK
```

即使字符串是 Redis 的基本值，您也可以使用它们执行一些有趣的操作。例如，一个是原子递增：

```
> set counter 100
OK
> incr counter
(integer) 101
> incr counter
(integer) 102
> incrby counter 50
(integer) 152
```

[INCR](../commands/INCR.md) 命令将字符串及诶为整数，将其加一，最后将获得的值设置为新值。类似的命令有 [INCRBY](../commands/INCRBY.md)、[DECR](../commands/DECR.md) 和 [DECRBY](../commands/DECRBY.md)。实际上，它们在内部就是同一个命令，只是看上去有点而不同。

[INCR](../commands/INCR.md) 是原子操作意味着什么呢？就是说即使多个客户端对同一个 key 发出 [INCR](../commands/INCR.md) 命令，也绝不会导致竞争的情况。例如如下情况永远不可能发生：客户端 1 和 客户端 2 同时读出 "10"，它们两个都对其加到 11，然后将新值设置为 11。最终的值一定是 12，read-increment-set 操作完成时，其他客户的不会在同一时间执行任何命令。

有许多用于操作字符串的命令。比如，[GETSET](../commands/GETSET.md) 命令将 key 设置为新值，返回旧值作为结果。你可以使用此命令，例如，如果你有一个系统，每次您的网站收到新访问者时都会使用 [INCR](../commands/INCR.md) 递增一个 Redis key。你可能希望每小时收集一次此信息，而不丢失每一个增量数据。你就可以使用 [GETSET](../commands/GETSET.md) 这个 key 并给其赋值 0 并读取原值。

在单个命令中设置或检索多个 key 的值，对于减少延迟也很有用。处于这个原因，有 [MSET](../commands/MSET.md) 和 [MGET](../commands/MGET.md) 命令：

```
> mset a 10 b 20 c 30
OK
> mget a b c
1) "10"
2) "20"
3) "30"
```

当使用 [MGET](../commands/MGET.md) 时，Redis 返回一个 value 数组。

---

### Altering and querying the key space

有些指令不是针对任何具体的类型定义的，而是用于和整个键空间(keyspace)交互的。因此，它们可被用于任何类型的 key。

例如，使用 [EXISTS](../commands/EXISTS.md) 命令返回 1 或 0 标识给定 key 的值是否存在，使用 [DEL](../commands/DEL.md) 命令可以删除 key 对应的值，无论值是什么。

```
> set mykey hello
OK
> exists mykey
(integer) 1
> del mykey
(integer) 1
> exists mykey
(integer) 0
```

从示例中，您还可以看到 [DEL](../commands/DEL.md) 本身如何返回 1 或 0，具体取决于 key 是否被删除（值存在）或者没有被删除（key对应的值不存在）。

与键空间相关的命令有很多，但以上两个是必不可少的，还有 [TYPE](../commands/TYPE.md) 命令，它返回存储在指定 key 上的值的种类：

```
> set mykey x
OK
> type mykey
string
> del mykey
(integer) 1
> type mykey
none
```

---

### Redis expires: keys with limited time to live

在继续讨论更复杂的数据结构之前，我们需要讨论另一个与 value 类型无关的 Redis 特性：**Redis expires**。基本上，你可以为 key 设置超时，这是一个有限的生存时间。当生存时间结束时，key 会自动销毁，就像用户使用 key 调用 [DEL](../commands/DEL.md) 命令一样。

关于 Redis expires 的一些简单信息：
- 可以使用秒或毫秒精度设置它们。
- 但是，过期时间的最小单位始终为 1 毫秒。
- 有关过期的信息被复制并保存在磁盘上，当您的 Redis 服务器保持停止状态时，时间实际上已经过去了（这意味着 Redis 会保存 key 的过期日期）。

设置过期很简单：

```
> set key some-value
OK
> expire key 5
(integer) 1
> get key (immediately)
"some-value"
> get key (after some time)
(nil)
```

key 在两次 [GET](../commands/GET.md) 调用之间消失了，因为第二次调用延迟了 5 秒以上。在上面的例子中，我们使用 [EXPIRE](../commands/EXPIRE.md) 来设置过期时间（它也可以用来为已经拥有的 key 设置不同的过期时间，比如可以使用 [PERSIST](../commands/PERSIST.md) 来删除过期时间并使 key 永久存储）。但是，我们也可以使用其他 Redis 命令创建带有过期时间的 key。例如使用 [SET](../commands/SET.md) 选项：

```
> set key 100 ex 10
OK
> ttl key
(integer) 9
```

上面的实例设置了一个字符串值为 100 的 key，其过期时间为 10s。稍后调用 [TTL](../commands/TTL.md) 命令以检查 key 的剩余生存时间。

为了以毫秒为单位设置和检查过期时间，请使用 [PEXPIRE](../commands/PEXPIRE.md) 和 [PTTL](../commands/PTTL.md) 命令，以及 [SET](../commands/SET.md) 选项的完整列表。

---

### Redis Lists

为了解释 List 数据类型，最好从一些理论开始，因为信息技术人员经常以不正确的方式使用属于 List。例如，"Python Lists" 并不是名称所表示的（Linked Lists），而是数组（实际上在 Ruby 中将相同的数据类型成为数组）。

从一个非常普遍的角度来看，List 只是一个有序元素的序列：10,20,1,2,3 是一个 list。但是使用 Array 实现的 List 的属性与使用 _Linked List_ 实现的 List 的属性非常不同。

Redis list 基于 Linked List 实现。这意味着即使在一个 list 中有数百万个元素，在头部或尾部添加一个元素的操作，其时间复杂度也是常数级别的。用 [LPUSH](../commands/LPUSH.md) 命令在是个元素的 list 头部添加新元素，和在千万元素的 list 头部添加新元素的速度相同。

那么，坏消息是什么？在数组实现的 list 中利用 *索引* 访问元素的速度极快（访问索引为恒定时间），而同样的操作在 linked list 实现的 list 上没有那么快（其中操作需要的工作量与访问元素的索引成正比）。

Redis Lists 是用链表实现的，因为对于数据库系统来说，能够以非常快的方式将元素添加到很长地列表中是至关重要的。另一个重要因素是，正如你将要看到的：Redis lists 能够在常数时间取得常数长度。

当快速访问大量元素的中间很重要时，可以使用不同的数据结构，称为 Sorted Set。本教程稍后将介绍 Sorted Set。

---

### First steps with Redis Lists

[LPUSH](../commands/LPUSH.md) 命令在列表的左侧（在头部）添加一个新元素，而 [RPUSH](../commands/RPUSH.md) 命令在列表的右侧（在尾部）添加一个新元素。最后 [LRANGE](../commands/LRANGE.md) 命令从列表中提取范围内的元素：

```
> rpush mylist A
(integer) 1
> rpush mylist B
(integer) 2
> lpush mylist first
(integer) 3
> lrange mylist 0 -1
1) "first"
2) "A"
3) "B"
```

注意：[LRANGE](../commands/LRANGE.md) 带有两个索引，一定范围的第一个和最后一个元素。这两个索引都可以为负数，来告知 Redis 从尾部开始计数，因此 -1 表示最后一个元素，-2 表示 list 中的倒数第二个元素，以此类推。

如你所见，[RPUSH](../commands/RPUSH.md) 将元素追加在列表右侧，而最终的 [LPUSH](../commands/LPUSH.md) 将元素追加在列表左侧。

这两个命令都是 _variadic commands_，这意味着您可以在一次调用中自由地将多个元素推送到列表中：

```
> rpush mylist 1 2 3 4 5 "foo bar"
(integer) 9
> lrange mylist 0 -1
1) "first"
2) "A"
3) "B"
4) "1"
5) "2"
6) "3"
7) "4"
8) "5"
9) "foo bar"
```

Redis 列表上定义的一个重要操作是 _pop elements_。弹出元素是同时从列表中检索元素和从列表中删除元素的操作。你可以从左侧和右侧弹出元素，类似于如何在列表的两侧 push 元素：

```
> rpush mylist a b c
(integer) 3
> rpop mylist
"c"
> rpop mylist
"b"
> rpop mylist
"a"
```

我们添加了三个元素并弹出了三个元素，所以在这个命令序列的末尾，列表是空的，没有更多的元素要弹出来。如果我们尝试弹出另一个元素，这就是我们得到的结果：

```
> rpop mylist
(nil)
```

Redis 返回一个 NULL 值以表示列表中没有元素。

---

### Common use cases for lists

Lists 对许多任务很有用，以下是两个非常有代表性的用例：
- 记住用户发布到社区网络的最新更新。
- 进程之间的通信，使用 consumer-producer 模式，其中 producer 将项目推送到列表中，consumer （通常是工作人员）消费这些项目并执行操作。Redis 有特殊的列表命令，使这个用例更加可靠和高效。

例如，流行的 Ruby 库 [resque](https://github.com/resque/resque) 和 [sidekiq](https://github.com/mperham/sidekiq) 都在后台使用 Redis 列表来实现后台作业。

流行的 Twitter 社交网络将用户发布的 [takes the latest tweets](http://www.infoq.com/presentations/Real-Time-Delivery-Twitter) 放入 Redis 列表。

为了逐步描述一个常见的用例，假设您的主页显示了在照片共享社交网络中发布的最新照片，并且您希望加快访问速度。
- 每次用户发布新照片时，我们都会使用 [LPUSH](../commands/LPUSH.md) 将其 ID 添加到列表中。
- 当用户访问主页时，我们使用 `LRANGE 0 9` 来获取最新发布的 10 个项目。

---

### Capped lists

在许多用例中，我们只想使用列表来存储最新的项目，无论它们是什么：社交网络更新、日志或其他任何东西。

Redis 允许我们使用列表作为上限集合，值记住最新的 N 项并使用 [LTRIM](../commands/LTRIM.md) 命令丢弃所有最旧的项。

[LTRIM](../commands/LTRIM.md) 命令类似于 [LRANGE](../commands/LRANGE.md)，但 **它不是显示指定范围的元素**，而是将此范围设置的元素为新列表的值。删除给定范围之外的所有元素。

一个例子会更清楚：

```
> rpush mylist 1 2 3 4 5
(integer) 5
> ltrim mylist 0 2
OK
> lrange mylist 0 -1
1) "1"
2) "2"
3) "3"
```

上面的 [LTRIM](../commands/LTRIM.md) 命令告诉 Redis 只获取从索引 0 到 2 的列表元素，其他元素都将被丢弃。

这允许一个非常简单但有用的模式：一起做 列表推送操作 + 列表修剪操作，以便添加一个新元素并丢弃超过限制的元素：

```
LPUSH mylist <some element>
LTRIM mylist 0 999
```

上述组合添加了一个新元素，并且只将 1000 个最新元素放入列表中。使用 [LRANGE](../commands/LRANGE.md)，您可以访问最重要的项目，而无需记住旧数据。

注意：虽然 [LRANGE](../commands/LRANGE.md) 在技术上是一个 O(N) 命令，但访问列表头部或尾部的小范围是一个常量时间操作。

---

### Blocking operations on lists

列表有一个特殊的特性，使他们适合实现队列，并且通常作为进程间通信系统的构建块：阻塞操作。

想象一下，你想通过一个流程将项目推送到列表中，并使用不同的流程来实际处理这些项目。这是常见的 producer/consumer 设置，可以通过以下简单方式实现：
- 要将项目推送到列表中，producer 调用 [LPUSH](../commands/LPUSH.md)。
- 为了从列表中提取/处理项目，消费者调用 [RPOP](../commands/RPOP.md)。

然而，有可能列表是空的，并且没有任何要处理的东西，所以 [RPOP](../commands/RPOP.md) 只返回 NULL。

在这种情况下，consumer 被迫等待一段时间并使用 [RPOP](../commands/RPOP.md) 重试。这称为轮询，这这种情况下不是一个好主意，因为它有几个缺点：
1. 强制 Redis 和客户端处理无用的命令（列表为空时的所有请求将不会完成任何实际工作，它们只会返回 NULL）。
2. 为项目的处理增加延迟，因为在工作人员收到 NULL 之后，它会等待一段时间。为了使延迟更小，我们可以在调用 [RPOP](../commands/RPOP.md) 之间减少等待，从而放大问题 1，即更多无用的 Redis 调用。

因此，Redis 实现了名为 [BRPOP](../commands/BRPOP.md) 和 [BLPOP](../commands/BLPOP.md) 的命令，它们是 [RPOP](../commands/RPOP.md) 和 [LPOP](../commands/LPOP.md) 的版本，如果列表为空，则能够阻塞它们：只有在列表中添加新元素或用户指定的超时时间到达时，它们才会返回给调用者。

这是我们可以在 worker 中使用的 [BRPOP](../commands/BRPOP.md) 调用的示例：

```
> brpop tasks 5
1) "tasks"
2) "do_something"
```

这意味着："等待列表任务中的元素，但如果 5 秒后没有元素可用则返回"。

请注意，您可以使用 0 作为超时来永远等待元素，也可以指定多个列表而不是一个，以便同时等待多个列表，并在第一个列表接收到元素时得到通知。

关于 [BRPOP](../commands/BRPOP.md) 的一些注意事项：

1. 客户端以有序的方式提供服务：第一个阻塞等待列表的客户端，当某个元素呗其他客户端推送时，首先被服务，以此类推。
2. 返回值与 [RPOP](../commands/RPOP.md) 不同：它是一个两个元素的数组，因为它还包含 key 的名称，因为 [BRPOP](../commands/BRPOP.md) 和 [BLPOP](../commands/BLPOP.md) 能够阻塞等待来自多个列表的元素。
3. 如果达到超时，则返回 NULL。

关于列表和阻塞操作，您还应该了解更多信息。我们建议您阅读以下内容：
- 可以使用 [LMOVE](../commands/LMOVE.md) 构建更安全的队列或轮换队列。
- 该命令还有一个阻塞变体，称为 [BLMOVE](../commands/BLMOVE.md)。

---

### Automatic creation and removal keys

到目前为止，在我们的示例中，我们从来没有在推送元素之前创建空列表，或者当它们不再有元素时删除空列表。Redis 负责在列表为空时删除 key，或者 key 不存在，我们正在尝试向其中添加元素时，创建一个空列表，例如，使用 [LPUSH](../commands/LPUSH.md)。

这不是特定于列表的，它适用于由多个元素组成的所有 Redis 数据类型 —— Streams、Sets、Sorted Sets、Hashes。

基本上，我们可以用三条规则来概括它的行为：
1. 当我们向一个聚合数据类型中添加元素时，如果目标 key 不存在，就在添加元素前创建空的聚合数据类型。
2. 当我们从聚合数据类型中移除元素时，如果值变为空的，key 自动被销毁。
3. 对一个空的 key 调用只读的命令，比如 [LLEN](../commands/LLEN.md)（返回列表的长度），或者一个删除元素的命令，将总是产生同样的结果。该结果和对一个空的聚合类型做同操作结果是一样的。

规则 1 示例：

```
> del mylist
(integer) 1
> lpush mylist 1 2 3
(integer) 3
```

但是，我们不存对存在但类型错误的 key 做操作：

```
> set foo bar
OK
> lpush foo 1 2 3
(error) WRONGTYPE Operation against a key holding the wrong kind of value
> type foo
string
```

规则 2 示例：

```
> lpush mylist 1 2 3
(integer) 3
> exists mylist
(integer) 1
> lpop mylist
"3"
> lpop mylist
"2"
> lpop mylist
"1"
> exists mylist
(integer) 0
```

所有的元素被弹出之后，key 不复存在。

规则 3 示例：

```
> del mylist
(integer) 0
> llen mylist
(integer) 0
> lpop mylist
(nil)
```

---

### Redis Hashes

Redis hashes 看起来就像是一个 "hash" 的样子，由键值对组成：

```
> hmset user:1000 username antirez birthyear 1977 verified 1
OK
> hget user:1000 username
"antirez"
> hget user:1000 birthyear
"1977"
> hgetall user:1000
1) "username"
2) "antirez"
3) "birthyear"
4) "1977"
5) "verified"
6) "1"
```

Hash 便于表示 _object_，实际上，你可以放入一个 hash 的字段数量没有限制（除了可用内存以外）。所以，你可以在你的应用中以不同的方式使用 hash。

[HMSET](../commands/HMSET.md) 指令设置 hash 中的多个字段，而 [HGET](../commands/HGET.md) 取回单个字段。[HMGET](../commands/HMGET.md) 和 [HGET](../commands/HGET.md) 类似，但返回 value 数组：

```
> hmget user:1000 username birthyear no-such-field
1) "antirez"
2) "1977"
3) (nil)
```

有些命令也可以对单个字段执行操作，例如 [HINCRBY](../commands/HINCRBY.md) ：

```
> hincrby user:1000 birthyear 10
(integer) 1987
> hincrby user:1000 birthyear 10
(integer) 1997
```

你可以在 [full list of hash commands in the documentation](https://redis.io/commands#hash) 找到。

值得注意的是，小的 hash（即一些具有小值的元素）在内存中以特殊方式编码，这使得它们非常节约内存。

---

### Redis Sets

Redis Set 是无序的字符串集合。[SADD](../commands/SADD.md) 命令想集合添加新元素。还可以对集合执行许多其他操作，例如测试给定元素是否已经存在，执行多个集合之间的交集、并集或差集，等待。

```
> sadd myset 1 2 3
(integer) 3
> smembers myset
1. 3
2. 1
3. 2
```

在这里，我向我的集合中添加了三个元素，并告诉 Redis 返回所有元素。正如您所看到的，它们没有排序 —— Redis 可以在每次调用时以任何顺序自由返回元素，因为与用户没有关于元素排序的合约。

Redis 有测试成员资格的命令。例如，检查元素是否存在：

```
> sismember myset 3
(integer) 1
> sismember myset 30
(integer) 0
```

"3" 是集合的成员，而 "30" 不是。

集合有利于表达对象之间的关系。例如，我们可以轻松地使用集合来实现标签。

一个简单的建模方式是，对每一个希望标记的对象使用 set。这个 set 包含和对象相关联的标签的 ID。

假设们想要给新闻打上标签。假设新闻 ID 1000 被打上的 1，2，5 和 77 四个标签，我们可以使用一个 set 把 tag ID 和新闻条目关联起来：

```
> sadd news:1000:tags 1 2 5 77
(integer) 4
```

但是，有时候我可能会需要相反的关系：所有被打上相同标签的新闻列表：

```
> sadd tag:1:news 1000
(integer) 1
> sadd tag:2:news 1000
(integer) 1
> sadd tag:5:news 1000
(integer) 1
> sadd tag:77:news 1000
(integer) 1
```

获取一个对象的所有 tag 是很方便的：

```
> smembers news:1000:tags
1. 5
2. 1
3. 77
4. 2
```

注意：在这个例子中，我们假设你有另一个数据结构，比如一个 Redis hash，把标签 ID 对应到标签名称。

使用 Redis 命令，我们可以轻易实现其他一些有用的操作。比如，我们可能需要一个含有 1，2，10 和 27 标签的对象的列表。我们可以用 [SINTER](../commands/SINTER.md) 命令来完成这件事。他获取不同 set 的交集。我们可以用：

```
> sinter tag:1:news tag:2:news tag:10:news tag:27:news
... results here ...
```

不只是可以取交集，还可以取并集、差集、获取随机元素，等等。

获取一个元素的命令是 [SPOP](../commands/SPOP.md) ，它很适合对特定问题建模。比如，要实现一个基于 web 的扑克牌游戏，您可能需要用 set 来表示一副牌。假设我们用一个字符的前缀来表示不同花色，`(C)lubs, (D)iamonds, (H)earts, (S)pades`：

```
sadd deck C1 C2 C3 C4 C5 C6 C7 C8 C9 C10 CJ CQ CK
   D1 D2 D3 D4 D5 D6 D7 D8 D9 D10 DJ DQ DK H1 H2 H3
   H4 H5 H6 H7 H8 H9 H10 HJ HQ HK S1 S2 S3 S4 S5 S6
   S7 S8 S9 S10 SJ SQ SK
   (integer) 52
```

现在，我们想要给每个玩家 5 张牌。[SPOP](../commands/SPOP.md) 命令删除一个随机元素，把它返回给客户端，因此它是完全合适的操作。

但是，如果我们直接针对我们的牌组调用他，在下一次游戏中我们需要再次填充一副牌，这可能并不理想。因此，首先，我们可以将存储在 `deck` key 中的集合复制到 `game:1:deck` key 中：

这是通过 [SUNIONSTORE](../commands/SUNIONSTORE.md) 实现的，它通常用于对多个集合取并集，并把结果存入另一个 set 中。但是，因为 set 的并集就是它本身，我可以这样复制我的牌：

```
> sunionstore game:1:deck deck
(integer) 52
```

现在，我已经准备好给 1 号玩家发五张牌：

```
> spop game:1:deck
"C6"
> spop game:1:deck
"CQ"
> spop game:1:deck
"D1"
> spop game:1:deck
"CJ"
> spop game:1:deck
"SJ"
```

一对 jacks，并不是很好...

现在是介绍 set 命令的好时机，该命令提供集合汇总元素的数量。这在集合论的上下文中通常称为集合的 _基数(cardinality)_，因此 Redis 命令称为 [SCARD](../commands/SCARD.md) 。

```
> scard game:1:deck
(integer) 47
```

数学计算：52 - 5 = 47。

当您只需要获取随机元素而不将它们从集合中删除时，有适合该任务的 [SRANDMEMBER](../commands/SRANDMEMBER.md) 命令。它还具有返回重复和非重复元素的能力。

---

### Redis Sorted sets

Sorted set 是一种类似于 set 和 hash 的混合数据类型。与集合一样，Sorted set 由唯一的、不重复的字符串元素组成，因此在某种意义上，Sorted set 也是一个集合。

然而，虽然集合内的元素没有排序，Sorted set 中的每个元素都与一个浮点值相关联，称为分数（这就是为什么类型也类似于 hash，因为每个元素都映射到一个值）。

此外，Sorted set 中的元素是按顺序获取的（因此它们不是按请求排序的，顺序是用于表示 Sorted set 的数据结构的特性）。它们根据以下规则排序：
- 如果 A 和 B 是具有不同分数的两个元素，如果 A.score \> B.score，则 A \> B。
- 如果 A 和 B 的分数完全相同。如果 A 字符串的 lexicographically 排序大于 B 字符串，则 A \> B。 A 和 B 字符串不能相等，因为 Sorted set 只有唯一元素。

让我们从一个简单的例子开始，添加一些选定的黑客名称作为 Sorted set 元素，他们的出生年份作为"分数"。

```
> zadd hackers 1940 "Alan Kay"
(integer) 1
> zadd hackers 1957 "Sophie Wilson"
(integer) 1
> zadd hackers 1953 "Richard Stallman"
(integer) 1
> zadd hackers 1949 "Anita Borg"
(integer) 1
> zadd hackers 1965 "Yukihiro Matsumoto"
(integer) 1
> zadd hackers 1914 "Hedy Lamarr"
(integer) 1
> zadd hackers 1916 "Claude Shannon"
(integer) 1
> zadd hackers 1969 "Linus Torvalds"
(integer) 1
> zadd hackers 1912 "Alan Turing"
(integer) 1
```

正如您所看到的，[ZADD](../commands/ZADD.md) 类似于 [SADD](../commands/SADD.md)，但需要一个额外的参数（放置在要添加的元素之前），即分数。[ZADD](../commands/ZADD.md) 也是可变参数，因此您可以自由指定多个分值对，即使在上面的实例中没有使用。

对于 Sorted set，返回一个按出生年份排序的黑客列表是微不足道的，因为实际上他们已经排序了。

实现说明：Sorted set 是通过双端口数据结构实现的，包括一个 skip list 和一个 hash table，所以每次我们添加一个元素，Redis 都会执行一个 O(log(N)) 操作。这很好，但是当我们要求 Sorted set 排序元素时，Redis 根本不需要做任何工作，他已经全部排序了：

```
> zrange hackers 0 -1
1) "Alan Turing"
2) "Hedy Lamarr"
3) "Claude Shannon"
4) "Alan Kay"
5) "Anita Borg"
6) "Richard Stallman"
7) "Sophie Wilson"
8) "Yukihiro Matsumoto"
9) "Linus Torvalds"
```

注意：0 和 -1 表示从元素索引 0 到最后一个元素（-1 在这里的作用就像在 [LRANGE](../commands/LRANGE.md) 命令的情况下一样）。

如果我想以相反的方式订购它们，从小到大怎么办？使用 [ZREVRANGE](../commands/ZREVRANGE.md) 而不是 [ZRANGE](../commands/ZRANGE.md)：

```
> zrevrange hackers 0 -1
1) "Linus Torvalds"
2) "Yukihiro Matsumoto"
3) "Sophie Wilson"
4) "Richard Stallman"
5) "Anita Borg"
6) "Alan Kay"
7) "Claude Shannon"
8) "Hedy Lamarr"
9) "Alan Turing"
```

也可以使用 `WITHSCORES` 参数返回分数：

```
> zrange hackers 0 -1 withscores
1) "Alan Turing"
2) "1912"
3) "Hedy Lamarr"
4) "1914"
5) "Claude Shannon"
6) "1916"
7) "Alan Kay"
8) "1940"
9) "Anita Borg"
10) "1949"
11) "Richard Stallman"
12) "1953"
13) "Sophie Wilson"
14) "1957"
15) "Yukihiro Matsumoto"
16) "1965"
17) "Linus Torvalds"
18) "1969"
```

---

### Operating on ranges

Sorted set 比这更强大。它们可以在范围内操作。让我们把所有出生到 1950 年的人都包括在内。我们使用 [ZRANGEBYSCORE](../commands/ZRANGEBYSCORE.md) 命令来做到这一点：

```
> zrangebyscore hackers -inf 1950
1) "Alan Turing"
2) "Hedy Lamarr"
3) "Claude Shannon"
4) "Alan Kay"
5) "Anita Borg"
```

我们要求 Redis 返回分数在负无穷和 1950 之间的所有元素（包括两个极端）。

也可以删除范围内的元素。然我们从 Sorted set 中删除所有在 1940 年到 1960 年之间出声的黑客：

```
> zremrangebyscore hackers 1940 1960
(integer) 4
```

[ZREMRANGEBYSCORE](../commands/ZREMRANGEBYSCORE.md) 可能不是最好的命令名称，但它可能非常有用，并返回已删除的元素的数量。

为 Sorted set 元素定义的另一个非常有用的操作是 get-rank 操作。可以获取一个元素在 Sorted set 中的位置是什么。

```
> zrank hackers "Anita Borg"
(integer) 4
```

考虑到元素按降序排序，[ZREVRANK](../commands/ZREVRANK.md) 命令也可用于获取排名。

---

### Lexicographical scores

在 Redis 2.8 及之后版本中，引入了一个新功能，允许按 lexicographically 顺序获取范围，假设 Sorted set 中的元素都以相同的分数插入（元素使用 C 的 `memcmp` 函数进行比较，因此可以保证没有排序规则，并且每个 Redis 实例都将回复相同的输出 ）。

lexicographical 范围操作的主要命令是：[ZRANGEBYLEX](../commands/ZRANGEBYLEX.md) 、[ZREVRANGEBYLEX](../commands/ZREVRANGEBYLEX.md) 、[ZREMRANGEBYLEX](../commands/ZREMRANGEBYLEX.md) 和 [ZLEXCOUNT](../commands/ZLEXCOUNT.md) 。

例如，让我们再次添加我们的著名黑客列表，但这次对所有元素使用零分：

```
> zadd hackers 0 "Alan Kay" 0 "Sophie Wilson" 0 "Richard Stallman" 0
  "Anita Borg" 0 "Yukihiro Matsumoto" 0 "Hedy Lamarr" 0 "Claude Shannon"
  0 "Linus Torvalds" 0 "Alan Turing"
```

由于 sorted set 的排序规则，它们已经按 lexicographically 顺序排序了：

```
> zrange hackers 0 -1
1) "Alan Kay"
2) "Alan Turing"
3) "Anita Borg"
4) "Claude Shannon"
5) "Hedy Lamarr"
6) "Linus Torvalds"
7) "Richard Stallman"
8) "Sophie Wilson"
9) "Yukihiro Matsumoto"
```

使用 [ZRANGEBYLEX](../commands/ZRANGEBYLEX.md) 我们可以要求 lexicographical 范围：

```
> zrangebylex hackers [B [P
1) "Claude Shannon"
2) "Hedy Lamarr"
3) "Linus Torvalds"
```

范围可以是包含的或不包含的（取决于第一个字符），也可以分别使用 + 和 - 字符串指定正无穷和负无穷。有关更多信息，请参阅文档。

这个特性很重要，因为它允许我们使用 Sorted set 作为通用索引。例如，如果你想通过 128 位无符号整数参数索引元素，您需要做的就是将元素添加到具有相同分数（例如 0）但具有 16 字节前缀的 Sorted set 中，其中包含 **128 位大端的位数**。由于大端的数字，当按 lexicographically 排序时（根据原始字节排序），您可以在 128 位空间中请求范围，并获得元素的值，丢弃前缀。

如果您想在更严肃的演示的上下文中查看该功能，请查看 [Redis autocomplete demo](http://autocomplete.redis.io/) 。

---

### Updating the score: leader boards

在切换到下一个话题之前，只是关于 Sorted set 的最后说明。Sorted set 的分数可以随时更新。仅针对已包含在 Sorted set 中的元素调用 [ZADD](../commands/ZADD.md) 将更新其分数（和位置），时间复杂度为 O(log(N))。因此，当有大量更新时，Sorted set 是合适的。

由于这个特性，一个常见的用例是排行榜。典型的应用程序是 Facebook 游戏，您可以将按高分排序的用户与获取排名操作结合起来，以显示前 N 个用户，以及用户在排行榜中的排名（例如，“你是这里的#4932 最高分”）。

---

### Bitmaps

Bitmap 不是实际的数据类型，而是在 String 类型上定义的一组面向 bit 的操作。由于字符串是二进制安全的 blob，并且它们的最大长度为 512MB，因此它们适合设置多大 2<sup>32</sup> 个不同的 bit。

bit 操作分为两组：常数时间和单个 bit 操作，例如将 bit 设置为 1 或 0，或获取其值，以及对 bit 组的操作，例如计算给定范围内的设置 bit 的数量（例如，人口计数）。

Bitmap 的最大优点之一是它们在存储信息时通常可以极大地节省空间。例如，在不同用户由增量用户 ID 表示的系统中，仅使用 512MB 内存就可以记住 40 亿用户的单个 bit 信息（例如，知道用户是否想要接收实时通讯）。

使用 [SETBIT](../commands/SETBIT.md) 和 [GETBIT](../commands/GETBIT.md) 命令设置和检索 bit ：

```
> setbit key 10 1
(integer) 1
> getbit key 10
(integer) 1
> getbit key 11
(integer) 0
```

[SETBIT](../commands/SETBIT.md) 命令的第一个参数是 bit number，即 1 或 0.如果寻址 bit 超出当前字符串长度，该命令会自动放大字符串。

[GETBIT](../commands/GETBIT.md) 金返回指定索引处的 bit 值。超出范围的 bit（寻址存储在目标 key 中的字符串长度之外的 bit）总被认为是 0。

有 3 个命令对 bit group 进行操作：
1. [BITOP](../commands/BITOP.md) 在不同的字符串之间执行按位运算。提供的操作是 AND、OR、XOR 和 NOT。
2. [BITCOUNT](../commands/BITCOUNT.md) 执行计数，报告设置为 1 的 bit 数量。
3. [BITPOS](../commands/BITPOS.md) 查找指定值为 0 或 1 的第一个 bit。

[BITPOS](../commands/BITPOS.md) 和 [BITCOUNT](../commands/BITCOUNT.md) 都能够对字符串的字节范围进行操作，而不是在字符串的整个长度上运行。以下是 [BITCOUNT](../commands/BITCOUNT.md) 调用的一个简单示例：

```
> setbit key 0 1
(integer) 0
> setbit key 100 1
(integer) 0
> bitcount key
(integer) 2
```

Bitmap 的常见用例是：
- 各种实时分析。
- 存储于对象 ID 关联的高性能高空间利用率的布尔信息。

例如，假设您想知道网站用户每天访问的最长连续记录。您从零开始计算天数，即您将网站公开的那一天，并在每次用户访问网站时使用 [SETBIT](../commands/SETBIT.md) 设置一个 bit。作为 bit 索引，您只需取当前 unix 时间，减去初始偏移量，然后除以一天中的秒数（通常为 3600*24）。

这样，对于每个用户，您都有一个包含每天访问信息的小字符串。使用 [BITCOUNT](../commands/BITCOUNT.md) 可以轻松获得给定用户访问网站的天数，而通过一些 [BITPOS](../commands/BITPOS.md) 调用，或简单地获取和分析客户端的 bitmap，可以轻松地计算最长的连续记录。

Bitmap 很容易拆分成多个 key，例如为了对数据集进行分片，因为通常最好避免使用大 key。要在不同的 key 之间拆分 bitmap，而不是将所有 bit 都设置为一个 key，一个简单的策略就是为每个 key存储 M bit，并使用 `bit-number/M` 获取 key 名和第 N 位，以使用 bit 在 key 内 `bit-number MOD M`。

---

### HyperLogLogs

HyperLogLog 是一种用于计算唯一事物的概率数据结构（从技术上讲，这称为估计集合的基数）。通常，计算唯一项需要使用与要计算的项数成正比的内存量，因为您需要记住过去已经看过的元素，以避免对它们进行多次计数。但是，有一组算法可以用内存换取精度：您以带有标准误差的估计度量结束，在 Redis 实现的情况下，该标准误差小于 1%。该算法的神奇之处在于，您不再需要使用与计数的项目数量成正比的内存量，而是可以使用恒定量的内存！在最坏的情况下为 12k 字节，或者如果您的 HyperLogLog（从现在起我们将它们称为 HLL）看到的元素很少，则更少。

Redis 中的 HLL 虽然在技术上是一种不同的数据结构，但被编码为 Redis 字符串，因此您可以调用 [GET](../commands/GET.md) 来序列化 HLL，并调用 [SET](../commands/SET.md) 将其反序列化回服务器。

从概念上讲，HLL API 就像使用 Sets 来完成相同的任务。您会将每个观察到的元素 [SADD](../commands/SADD.md) 放入一个集合中，并使用 [SCARD](../commands/SCARD.md) 检查集合内元素的数量，这些元素是唯一的，因为 [SADD](../commands/SADD.md) 不会重新添加现有元素。

虽然您并没有真正向 HLL 添加项目，因为数据结构只包含不包含实际元素的状态，API 是相同的：
- 每次看到一个新元素时，就使用 [PFADD](../commands/PFADD.md) 将其添加到计数中。
- 每次您想要检索到目前为止使用 [PFADD](../commands/PFADD.md) 添加的唯一元素的当前近似值时，您都可以使用 [PFCOUNT](../commands/PFCOUNT.md) 。


```
> pfadd hll a b c d
(integer) 1
> pfcount hll
(integer) 4
```

此数据结构的用例示例是计算用户每天在搜索表单中执行的唯一查询。

Redis 也能够执行 HLL 的并集，请查看 [full documentation](https://redis.io/commands#hyperloglog) 以获取更多信息。

---

### Other notable features

Redis API 中还有其他一些不能在本文档的上下文中探讨的重要内容，但值得您注意：
- 可以 [iterate the key space of a large collection incrementally](../commands/SCAN.md) 。
- 可以运行 [Lua scripts server side](../commands/EVAL.md) 以改善延迟和带宽。
- Redis 同样是一个 [Pub-Sub server](pubsub.md) 。

---

### Learn more

本教程并不完整，仅涵盖了 API 的基础知识。阅读 [command reference](../commands) 以发现更多信息。

Thanks for reading, and have fun hacking with Redis!