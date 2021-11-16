## XDEL key ID [ID ...]

    起始版本：5.0.0.
    时间复杂度：无论流大小如何，对于要在流中删除的每个单个项目都是 O(1)。

从流中删除指定的条目，并返回删除的条目数量。在流中不存在某些指定的 ID 的情况下，返回的数量可能与传递的 ID 数量不同。

通常，你可能将 Redis Stream 想象成为一个 append only 的数据结构，但是 Redis Stream 是存在于内存中的，所以我们也可以删除条目。这也许会有用，例如，为了遵守特定的隐私策略。

---

### Understanding the low level details of entries deletion

Redis streams 以一种使其内存高效的方式表示：使用基数树来索引包含线性数十个 Stream 条目的宏节点。通常，当你从 Stream 中删除一个条目的时候，条目并没有 _真正_ 被逐出，只是被标记为删除。

最终，如果宏节点中的所有条目都被标记为删除，则会销毁整个节点，并回收内存。这意味着如果你从 Stream 里删除大量的条目，比如超过 50% 的条目，则每一个条目的内存占用可能会增加，因为 Stream 将会开始变得碎片化。然而，stream 的表现将表示不变。

在 Redis 未来的版本中，当一个宏节点内删除条目达到一定数量的时候，我们有可能会触发节点垃圾回收机制。目前，根据我们队这种数据结构的预期用途，还不太适合增加这样的复杂度。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：实际删除的条目数量。

---

### Examples

```
> XADD mystream * a 1
1538561698944-0
> XADD mystream * b 2
1538561700640-0
> XADD mystream * c 3
1538561701744-0
> XDEL mystream 1538561700640-0
(integer) 1
127.0.0.1:6379> XRANGE mystream - +
1) 1) 1538561698944-0
   2) 1) "a"
      2) "1"
2) 1) 1538561701744-0
   2) 1) "c"
      2) "3"
```