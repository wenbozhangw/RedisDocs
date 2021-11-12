## ZRANGE key min max [BYSCORE|BYLEX] [REV] [LIMIT offset count] [WITHSCORES]

    起始版本：1.2.0.
    时间复杂度：O(log(N) + M)，其中 N 是 Sorted Set 中的元素数，M 是返回的元素数。

返回存储在 Sorted Set `<key>` 中的指定范围的元素。

[ZRANGE](ZRANGE.md) 可以执行不同类型的返回查询：by index(rank), by the score, or by lexicographical order。

从 Redis 6.2.0 开始，此命令可以替换以下命令：[ZREVRANGE](ZREVRANGE.md)、[ZRANGEBYSCORE](ZRANGEBYSCORE.md)、[ZREVRANGEBYSCORE](ZREVRANGEBYSCORE.md)、[ZRANGEBYLEX](ZRANGEBYLEX.md)、[ZREVRANGEBYLEX](ZREVRANGEBYLEX.md)。

---

### Common behavior and options

元素的顺序从最低分到最高分。具有相同分数的元素按字典顺序排列。

可选的 `REV` 参数颠倒排序，因此元素从最高分到最低分排序，并且分数关系通过 reverse lexicographical 排序来解决。

可选的 `LIMIT` 参数可用于从匹配元素中获取子范围（类似于 SQL 中的 SELECT offset, count）。负的 `<count>` 返回 `<offset>` 中的所有元素。请记住，如果 `<offset>` 很大，则需要为 `<offset>` 元素遍历已排序的集合，然后才能找到要返回的元素，这可能会增加 O(N) 时间复杂度。

可选的 `WITHSCORES` 参数可以返回元素对应的分数。返回的列表包含 `value1,score1,...,valueN,scoreN` 而不是 `value1,...,valueN`。客户端可以自由返回更适合的数据类型（建议：带有 (value, score) arrays/tuples 的数组）。

---

### Index ranges

默认情况下，该命令执行索引范围查询。`<min>` 和 `<max>` 参数表示从零开始的索引，其中 0 是第一个元素，1 是下一个元素，以此类推。这些参数指定一个**全包含的区间**，例如，ZRANGE myzset 0 1 将返回 Sorted Set 的第一个和第二个元素。 

索引也可以是负数，表示从 Sorted Set 末尾的偏移量，-1 是 Sorted Set 的最后一个元素，-2 是倒数第二个元素，以此类推。

超出范围的索引不会产生错误。

如果 `<min>` 大于 Sorted Set 的结束索引或 `<max>`，则返回一个空列表。

如果 `<max>` 大于 Sorted Set 的结束索引，Redis 将使用 Sorted Set 的最后一个元素。

---

### Score ranges

当提供 `BYSCORE` 选项时，该命令的行为类似于 [ZRANGEBYSCORE](ZRANGEBYSCORE.md) 并返回 Sorted Set 中的元素范围，其分数等于或介于 `<min>` 和 `<max>` 之间。

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

### Lexicographical ranges

当使用 `BYLEX` 选项时，该命令的行为类似于 [ZRANGEBYLEX](ZRANGEBYLEX.md) 并返回 `<min>` 和 `<max>` lexicographical closed 范围之间的 Sorted Set 元素。

请注意，lexicographical 顺序依赖于具有相同分数的所有元素。当元素具有不同分数时，回复是未指定的。

有效的 `<min>` 和 `<max>` 必须以 `(` 或 `[` 开头，以分别指定范围区间是不包含还是包含。

`+` 或 `-`，`<min>` 和 `<max>` 的特殊值分别表示正或负无穷字符串，因此例如命令 **ZRANGEBYLEX myzset - +** 保证返回 Sorted Set 中的所有元素，前提是所有元素都具有相同的分数。

---

#### Lexicographical comparison of strings

字符串作为二进制字节数组进行比较。由于 ASCII 字符集的指定方式，这意味着通常这也具有以明显的字典方式比较普通的 ASCII 字符的效果。但是如果使用非纯 ASCII 字符串（例如 utf8 字符串），则情况并非如此。

但是，用户可以对编码字符串应用转换，以便插入到 Sorted Set 中的元素的第一部分将按照用户对特定应用程序的要求进行比较。例如，如果我想添加会以不区分大小写的方式进行比较的字符串，但查询时仍然想检索真实的大小写，我可以通过以下方式添加字符串：

```
ZADD autocomplete 0 foo:Foo 0 bar:BAR 0 zap:zap
```

由于每个元素中的第一个标准化部分（在冒号字符之前），我们强制进行给定的比较。但是，在使用 **ZRANGE ... BYLEX** 查询范围后，应用程序可以像用户显示字符串的第二部分，在冒号之后。

比较的二进制特性允许使用 Sorted Set 作为通用索引，例如，元素的第一部分可以是 64 位大端数。由于大端数字在初始位置具有最高有效字节，因此而仅仅比较将使用数字进行比较。这可用于对 64 位值试试范围查询。如下例所示，在前 8 个字节之后，我们可以存储我们正在索引的元素的值。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) ：指定范围内的元素列表（如果给出了 `WITHSCORES` 选项，则可以选择带有它们的分数）。

---

### History

- &gt;= 6.2：增加 `REV`, `BYSCORE`, `BYLEX`, `LIMIT` 选项。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZADD myzset 3 "three"
(integer) 1
redis> ZRANGE myzset 0 -1
1) "one"
2) "two"
3) "three"
redis> ZRANGE myzset 2 3
1) "three"
redis> ZRANGE myzset -2 -1
1) "two"
2) "three"
redis> 
```

以下使用 `WITHSCORES` 的示例显示该命令如何始终返回一个数组，但这次填充了 `element_1, score_1, element_2, score_2, ..., element_N, score_N`。

```
redis> ZRANGE myzset 0 1 WITHSCORES
1) "one"
2) "1"
3) "two"
4) "2"
redis> 
```

此示例显示如何查询按分数排序的集合，不包括值 1 到无穷大，仅返回结果的第二个元素：

```
redis> ZRANGE myzset (1 +inf BYSCORE LIMIT 1 1
1) "three"
redis> 
```