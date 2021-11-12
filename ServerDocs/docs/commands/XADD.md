## XADD key [NOMKSTREAM] [MAXLEN|MINID [=|~] threshold [LIMIT count]] *|ID field value [field value ...]

    起始版本：5.0.0。
    时间复杂度：当添加新的条目为O(1)，修剪时为O(N)，其中 N 是被逐出的条目数量。

将指定的流条目追加到指定 key 的流中。如果 key 不存在，作为运行这个命令的副作用，将使用流的条目自动创建 key。可以使用 `NOMKSTREAM` 选项禁止流 key 的创建。

一个条目是由一组键值对组成的，它基本上是一个小的字典。键值对以用户给定的顺序存储，并且读取流的命令，比如 [XRANGE](XRANGE.md) 或 [XREAD](XREAD.md) 可以保证按照通过 [XADD](XADD.md) 添加的顺序返回。

[XADD](XADD.md) 是唯一可以向流中添加数据的 Redis 命令，但还有其他命令，例如 [XDEL](XDEL.md) 和 [XTRIM](XTRIM.md)，可以从流中删除数据。

---

### Specifying a Stream ID as an argument

流条目 ID 标识流内的给定条目。

如果指定的 ID 参数是字符 `*` (星号 ASCII 字符)，[XADD](XADD.md) 命令会自动为您生成一个唯一 ID。但是，也可以指定一个良好格式的 ID，以便新的条目以指定的 ID 准确存储，虽然仅在极少情况下有用。

ID 是由 - 隔开的两个数字组成：

```
1526919030474-55
```

两个部分数字都是 64 位的，当自动生成 ID 时，第一部分是生成 ID 的 Redis 实例的毫秒格式的 Unix 时间。第二部分只是一个序列号，以及是用来区分同一毫秒内生成的 ID 的。

ID 保证始终是递增的：如果比较刚插入的条目的 ID，它将大于其他任何过期的 ID，因此条目在流中是完全排序的。为了保证这个特性，如果流中当前最大的 ID 的时间大于实例的当前本地时间，将会使用前者，并将 ID 的序列部分递增。例如，本地时钟回调了，或者故障转移之后新主机具有不同的绝对时间，则可能会发生这种情况。

当用户为 [XADD](XADD.md) 命令指定显式 ID 时，最小有效的 ID 是 0-1，并且用户必须指定一个比当前流中的任何 ID 都要大的 ID，否则命令将失败。通常使用特定 ID 仅在您有另一个系统生成唯一 ID （例如SQL表），并且您确实希望 Redis 流ID与该另一个系统的ID匹配时才有用。

---

### Capped streams

[XADD](XADD.md) 包含与 [XTRIM](XTRIM.md) 命令相同的语义 —— 有关更多信息，请参阅其文档页面。[XADD](XADD.md) 命令允许添加新条目并通过对 [XADD](XADD.md) 的单个调用来检查流的大小，从而使用任意阈值有效地限制流。尽管精确修剪是可能的并且是默认设置，但由于流的内部表示，使用几乎精确修剪（`~`参数）使用 [XADD](XADD.md) 条件条目和修剪流更有效。

例如，以下列形式调用 [XADD](XADD.md) ：

```
XADD mystream MAXLEN ~ 1000 * ... entry fields here ...
```

将添加一个新条目，但也会驱逐旧条目，以便流将仅包含 1000 个条目，或最多几十个条目。

---

### Additional information about streams

有关 Redis 流的更多信息，请查看我们的 [introduction to Redis Streams document](/docs/topics/streams-intro.md) 。

---

### Return Value

[Bulk string reply](/docs/topics/protocol.md#resp-bulk-strings)，特定的：

该命令返回添加的条目的ID。如果 * 作为 ID 参数传递，则 ID 是自动生成的，否则该命令金返回用户在插入期间指定的相同的 ID。

---

### History

- &gt;= 6.2 : 增加 **NOMKSTREAM** 选项，**MINID** 修剪策略和 **LIMIT** 选项。

---

### Examples

```
redis> XADD mystream * name Sara surname OConnor
"1632819333935-0"
redis> XADD mystream * field1 value1 field2 value2 field3 value3
"1632819333937-0"
redis> XLEN mystream
(integer) 2
redis> XRANGE mystream - +
1) 1) "1632819333935-0"
   2) 1) "name"
      2) "Sara"
      3) "surname"
      4) "OConnor"
2) 1) "1632819333937-0"
   2) 1) "field1"
      2) "value1"
      3) "field2"
      4) "value2"
      5) "field3"
      6) "value3"
redis> 
```