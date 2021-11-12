## XTRIM key MAXLEN|MINID [=|~] threshold [LIMIT count]

    起始版本：5.0.0。
    时间复杂度：O(N)，其中 N 是被驱逐条目的数量。然而，常量时间非常小，因为条目组织在包含多个条目的宏节点中，这些条目可以通过一次释放释放。

如果需要，[XTRIM](XTRIM.md) 通过驱逐较旧的条目（具有较低 ID 的条目）来修剪流。

可以使用以下策略之一来修剪流：
- `MAXLEN`: 只要流的长度超过指定的阈值，就会逐出条目，其中阈值是一个正整数。
- `MINID` : 驱逐 ID 低于阈值的条目，其中阈值是流 ID。

例如，这会将流修剪为最新的 1000 个条目：

```
XTRIM mystream MAXLEN 1000
```

而在本例中，所有 ID 低于 649085820-0 的条目都将被驱逐：

```
XTRIM mystream MINID 649085820
```

默认情况下，或者当提供可选的 `=` 参数时，该命令执行精确修剪。

根据策略，精确修剪意味着：
- `MAXLEN` : 修剪后的流的长度将恰好是其原始长度和指定阈值之间的最小值。
- `MINID` : 流中最旧的 ID 将恰好是其原始最旧 ID 和指定阈值之间的最小值。

---

### Nearly exact trimming

由于精确修剪可能需要 Redis 服务器的额外工作，因此可以提供可选的 `~` 参数以使其更高效。

例如：

```
XTRIM mystream MAXLEN ~ 1000
```

`MAXLEN` 策略和阈值之间的 `~` 意味着用户正在请求修剪流，使其长度至少为阈值，但可能略多。在这种情况下，Redis 会在尅获得性能时提前停止修剪（例如，当无法删除数据结构中的整个宏节点时）。这使得修剪的效率更高，这通常使您想要的，尽管在修剪后，流可能有几十个超过阈值的额外条目。

使用 `~` 时，控制命令完成的工作量的另外一种方法是 `LIMIT` 子句。使用时，它指定将被逐出的最大条目数。当未指定 `LIMIT` 和 `count` 时，默认值 100 * 宏节点(macro node)中的条目数将被隐式用作计算。将值 0 指定为 count 将完全禁用限制机制。

---

### Return Value

[Integer reply](/docs/topics/protocol.md#resp-integers) : 该命令返回从流中删除的条目数。

---

### History

- &gt;= 6.2 : 增加 `MINID` 裁剪策略和 `LIMIT` 选项。

---

### Examples

```
redis> XADD mystream * field1 A field2 B field3 C field4 D
"1632818360317-0"
redis> XTRIM mystream MAXLEN 2
(integer) 0
redis> XRANGE mystream - +
1) 1) "1632818360317-0"
   2) 1) "field1"
      2) "A"
      3) "field2"
      4) "B"
      5) "field3"
      6) "C"
      7) "field4"
      8) "D"
redis> 
```