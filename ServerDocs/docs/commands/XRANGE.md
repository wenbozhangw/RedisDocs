## XRANGE key start end [COUNT count]

    起始版本：5.0.0。
    时间复杂度：O(N)，其中 N 是返回的元素数量。如果 N 是常数（例如总是用 COUNT 要求前 10 个元素），你可以认为它是 O(1)。

该命令返回流中满足给定 ID 范围的条目。

范围由最小和最大 ID 指定。所有 ID 在指定的两个 ID 之间或与其中一个 ID 相等（闭合区间）的条目将会被返回。

[XRANGE](XRANGE.md) 命令有许多用途：
- 返回特定时间范围的条目。这是可能的，因为流的ID 是[related to time](/docs/topics/streams-intro.md)。
- 增量迭代，每次迭代只返回几个条目。但它在语义上比 [SCAN](SCAN.md) 函数族强大很多。
- 从流中获取单个条目，提供要获取两次的条目的 ID：作为查询间隔的开始和结束。

该命令还有一个倒序命令，以相反的顺序返回条目，叫做 [XREVRANGE](XREVRANGE.md)，除了返回顺序相反以外，它们是完全相同的。

---

### - and + special IDs

特殊的ID `-` 和 `+` 分别表示流中可能的最小 ID 和最大 ID，因此，以下命令将会返回流中的每一个条目：

```
> XRANGE somestream - +
1) 1) 1526985054069-0
   2) 1) "duration"
      2) "72"
      3) "event-id"
      4) "9"
      5) "user-id"
      6) "839248"
2) 1) 1526985069902-0
   2) 1) "duration"
      2) "415"
      3) "event-id"
      4) "2"
      5) "user-id"
      6) "772213"
... other entries here ...
```

`-` ID 实际上与指定 0-0 完全一样，而 `+` 则相当于 `18446744073709551615-18446744073709551615`，但是它们更适合输入。

---

### Incomplete IDs

流的 ID 由两部分组成，一个 Unix 毫秒时间戳和一个为同一毫秒插入的序列号。使用 [XRANGE](XRANGE.md) 仅指定 ID 的第一部分是可能的，即毫秒时间部分，如下面的例子所示：

```
> XRANGE somestream 1526985054069 1526985055069
```

在这种情况下，[XRANGE](XRANGE.md) 将会使用 **-0** 自动补全开始 ID，以及使用 **-18446744073709551615** 自动补全结束 ID，以便返回所有在两个毫秒值之间生成的条目。这同样意味着，重复两个相同的毫秒时间，我们将会得到在这一毫秒内产生的所有条目，因为序列号范围将从 0 到最大值。

以这种方式使用 [XRANGE](XRANGE.md) 作用范围查询命令已在指定时间内获取条目。这非常方便，一边访问流中过去时间的历史记录。

---

### Exclusive ranges

默认情况下，范围是闭区间（包括），这意味着回复可以包含 ID 与查询的开始和结束间隔匹配的条目。可以通过在 ID 前面加上字符 `(`。 这对于迭代流很有用，如下所述。

---

### Returning a maximum number of entries

使用 **COUNT** 选项可以减少返回的条目数量。这是一个非常重要的特性，虽然他看起来很边缘，因为它允许例如模型操作，比如给我大于或等于以下ID的条目：

```
> XRANGE somestream 1526985054069-0 + COUNT 1
1) 1) 1526985054069-0
   2) 1) "duration"
      2) "72"
      3) "event-id"
      4) "9"
      5) "user-id"
      6) "839248"
```

在上面的例子中，条目 1526985054069-0 存在，否则服务器将发送给我们下一个条目。使用COUNT也是使用 [XRANGE](XRANGE.md) 作为迭代器的基础。

---

### Iterating a stream

为了迭代流，我们可以如下进行。让我们假设每次迭代我们需要两个元素。我们开始获取前两个元素，这是很简单的：

```
> XRANGE writers - + COUNT 2
1) 1) 1526985676425-0
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

然后，不是从 - 再次开始迭代，我们使用前一次 [XRANGE](XRANGE.md) 调用返回的最后的条目 ID 作为范围的闭区间（exclusive interval）开始。

最后一个条目的 ID 是 1526985685298-0 ，因此我们只需要前缀使用一个 '('，然后继续我们的迭代：

```
> XRANGE writers (1526985685298-0 + COUNT 2
1) 1) 1526985691746-0
   2) 1) "name"
      2) "Toni"
      3) "surname"
      4) "Morrison"
2) 1) 1526985712947-0
   2) 1) "name"
      2) "Agatha"
      3) "surname"
      4) "Christie"
```

以此类推。最终，这将允许访问流中的所有条目。很明显，我们可以从任意 ID 开始迭代，或者甚至从特定的时间开始，通过提供一个不完整的开始 ID。此外，我们可以限制迭代到一个给定的 ID 或时间，通过提供一个结束ID或不完整ID而不是 +。

[XREAD](XREAD.md) 命令同样可以迭代流。[XREVRANGE](XREVRANGE.md) 命令可以反向迭代流，从较高的 ID（或时间）到较低的ID（或时间）。

#### Iterating with earlier versions if Redis

虽然 exclusive 范围区间仅在 Redis 6.2 中可用，但仍然可以在早期版本中使用类似的流迭代模式。您开始以与上述相同的方式从流中获取以获取第一个条目。

对于后续调用，您需要一边吃方式提前返回最后一个条目的 ID。大多数 Redis 客户端应该抽象这个细节，但如果需要，是吸纳也可以在应用程序中。在上面的示例中，这意味着将 1526985685298-0 的序列从 0 增加到 1.因此，第二个调用将是：

```
> XRANGE writers 1526985685298-1 + COUNT 2
1) 1) 1526985691746-0
   2) 1) "name"
      2) "Toni"
...
```

另外请注意，一旦最后一个 ID 的序列部分等于 18446744073709551615，您将需要增加时间戳并将序列部分重置为 0。例如，增加 ID 1526985685298-18446744073709551615 应该得到 1526985685299-0。

对称模式使用与使用 [XREVRANGE](XREVRANGE.md) 迭代流。唯一的区别是客户端需要为后续调用递减 ID。递减序列部分为0的ID时，时间戳需要递减1，序列设置为18446744073709551615。

---

### Fetching single items

如果你爱查找一个 **XGET** 命令，你将会失望，因为 [XRANGE](XRANGE.md) 实际上就是从流中获取单个条目的方式。所以你需要做的，就是在 XRANGE 的参数中指定 ID 两次：

```
> XRANGE mystream 1526984818136-0 1526984818136-0
1) 1) 1526984818136-0
   2) 1) "duration"
      2) "1532"
      3) "event-id"
      4) "5"
      5) "user-id"
      6) "7782813"
```

---

### Additional information about streams

有关 Redis 流的更多信息，请查看我们的 [introduction to Redis Streams document](/docs/topics/streams-intro.md) 。

---

### Return Value

[Array reply](/docs/topics/protocol.md#resp-arrays)，特定的：

该命令返回 ID 与指定范围匹配的条目。返回的条目是完整的，这意味着 ID 和所有组成条目的字段都将返回。此外，返回的条目及其字段和值的顺序与使用 [XADD](XADD.md) 添加它们的顺序完全一致。

---

### History

- &gt;= 6.2 : 增加 exclusive 范围。

---

### Examples

```
redis> XADD writers * name Virginia surname Woolf
"1632821675635-0"
redis> XADD writers * name Jane surname Austen
"1632821675635-1"
redis> XADD writers * name Toni surname Morrison
"1632821675636-0"
redis> XADD writers * name Agatha surname Christie
"1632821675636-1"
redis> XADD writers * name Ngozi surname Adichie
"1632821675636-2"
redis> XLEN writers
(integer) 5
redis> XRANGE writers - + COUNT 2
1) 1) "1632821675635-0"
   2) 1) "name"
      2) "Virginia"
      3) "surname"
      4) "Woolf"
2) 1) "1632821675635-1"
   2) 1) "name"
      2) "Jane"
      3) "surname"
      4) "Austen"
redis> 
```