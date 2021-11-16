## SRANDMEMBER key [count]

    自 1.0.0 起可用。
    时间复杂度：没有 count 参数为 O(1)，否则为 O(N)，其中 N 是传递的 count 值。

仅使用 key 参数调用，那么随机返回 key 集合中的一个元素。

如果提供的 count 参数为正数，则返回 **distinct elements** 的数组。数组的长度是 count 或集合的基数 ([SCARD](scard.md))，以较低者为准。

如果使用负数 count 调用，则行为会发生变化，并且该命令可以**多次返回相同的元素**。在这种情况下，返回元素的数量是指定 count 的绝对值。

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 如果没有额外的 count 参数，该命令会返回一个随机选择的元素的 Bulk Reply，或者当 key 不存在时，返回 nil。

[Array reply](../topics/protocol.md#resp-arrays) : 当传递额外的 count 参数时，该命令返回一个元素数组，或者当 key 不存在时返回一个空数组。


### Examples

```
redis> SADD myset one two three
(integer) 3
redis> SRANDMEMBER myset
"one"
redis> SRANDMEMBER myset 2
1) "two"
2) "one"
redis> SRANDMEMBER myset -5
1) "two"
2) "one"
3) "three"
4) "two"
5) "one"
redis>
```

### History

- &gt;= 2.6.0 : 增加可选的 count 参数。

### Specification of the behavior when count is passed

当 count 参数为正数时，此命令的行为如下：
- 不返回重复的元素。
- 如果 count 大于集合的基数，则该命令金返回整个集合而没有附加元素。
- 恢复中元素的顺序并不是真正随机的，因此如果需要，由客户端来调整它们。

当 count 参数为负数时，行为变化如下：
- 元素可能是重复的。
- 如果集合为空（不存在的 key ），则始终返回精确 count 的元素或空数组。
- 回复中元素的顺序是真正随机的。

### Distribution of returned elements

注意：本节仅与 Redis 5 或更低版本相关，因为 Redis 6 实现了更公平的算法。

当集合中的元素数量很少时，返回元素分布远不够完美，这是因为我们使用了一个近似随机元素函数，它并不能保证良好的分布。

所使用的算法（在 dict.c 中实现）对哈希表桶进行采样已找到非空桶。一旦找到非空桶，由于我们在哈希表的视线中使用了链接法，因此会检查桶中的元素数量，并且选出一个随机元素。

这意味着，如果你在整个哈希表中有两个非空桶，其中一个有三个元素，另一个只有一个元素，那么其桶中单独存在的元素将以更高的概率返回。