## ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]

    起始版本：1.0.5.
    时间复杂度：O(log(N) + M)，其中 N 是 Sorted Set 中的元素数，M 是返回的元素数。如果 M 是常数（例如，总是用 `LIMIT` 要求前 10 个元素），你可以认为它是 O(log(N))。

返回 key 对应的 Sorted Set 中的所有在 `<min>` 和 `<max>` 之间的元素（包括分数等于`<min>` 和 `<max>`的元素）。元素是根据分数从低到高排序的。

具有相同分数的元素按照 lexicographical 排序返回（这是根据 Redis 的 Sorted Set 实现的情况而定，并不需要进一步计算）。

根据 Redis 6.2.0，此命令被视为已弃用。请在新代码中使用带有 `BYSCORE` 参数的 [ZRANGE](ZRANGE.md) 命令。

可选的 `LIMIT` 参数可用于仅获取匹配元素的范围（类似于 SQL 中的 SELECT LIMIT offset,count）。负的 `<count>` 返回 `<offset>` 中的所有元素。请记住，如果 `<offset>` 很大，则需要为 `<offset>` 元素遍历已排序的集合，然后才能找到要返回的元素，这可能会增加 O(N) 时间复杂度。

可选的 `WITHSCORES` 参数使命令返回元素及其分数，而不是单独返回元素。此选项自 Redis 2.0 起可用。

---

### Exclusive intervals and infinity

`<min>` 和 `<max>` 可以是 `-inf` 和 `+inf`，分别表示负无穷大和正无穷大。这意味着您不需要知道 Sorted Set 中的最高或最低分数来获取从某个分数开始或到某分数的所有元素。

默认情况下，`<min>` 和 `<max>` 指定的分数区间是 closed (inclusive)。可以通过在分数前加上字符`(`。

例如：
```
ZRANGE zset (1 5 BYSCORE
```

将会返回所有 `1 < score <= 5` 的元素。

```
ZRANGE zset (5 (10 BYSCORE
```

将会返回所有 `5 < score < 10` （不包含 5 和 10）的元素。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：指定分数范围的元素列表（分数可选）。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZRANGEBYSCORE myzset -inf +inf
1) "one"
2) "two"
3) "three"
redis> ZRANGEBYSCORE myzset 1 2
1) "one"
2) "two"
redis> ZRANGEBYSCORE myzset (1 2
1) "two"
redis> ZRANGEBYSCORE myzset (1 (2
(empty list or set)
redis> 
```

---

### Pattern: weighted random selection of an element

通常 [ZRANGEBYSCORE](ZRANGEBYSCORE.md) 仅用于获取分数为整数索引键的范围内的元素，但是可以使用该命令做不那么明显的事情。

例如，在实现 Markov chains 和 其他算法时的一个场景问题是从集合中随机选择一个元素，但不同的元素可能具有不同的权重，这会改变它们被选中的可能性。

这是我们如何使用此命令来安装这样的算法：

假设您有元素 A、B 和 C，其权重分别为 1、2 和 3。您计算权重之和，即 `1+2+3 = 6`

此时，您使用此算法将所有元素添加到一个 Sorted Set 中：

```
SUM = ELEMENTS.TOTAL_WEIGHT // 6 in this case.
SCORE = 0
FOREACH ELE in ELEMENTS
    SCORE += ELE.weight / SUM
    ZADD KEY SCORE ELE
END
```

这意味着您设置：

```
A to score 0.16
B to score .5
C to score 1
```

由于这设计近似值，为了避免 C 将 0.998 设置为 1，我们只修改上述算法以确保最后一个分数为 1.

此时，每次你想得到一个加权的随机元素时，只需计算一个 0 到 1 之间的随机数（这就像在大多数语言中调用 rand() 一样），所以你可以这样做：

```
RANDOM_ELE = ZRANGEBYSCORE key RAND() +inf LIMIT 0 1
```