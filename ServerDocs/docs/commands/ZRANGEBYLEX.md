## ZRANGEBYLEX key min max [LIMIT offset count]

    起始版本：2.8.9.
    时间复杂度：O(log(N) + M)，其中 N 是 Sorted Set 中的元素数，M 是返回的元素数。如果 M 是常数（例如，总是用 `LIMIT` 要求前 10 个元素），你可以认为它是 O(log(N))。

当 Sorted Set 中的所有元素都以相同的分数插入时，为了强制按字典顺序排列，此命令返回 key 对应的 Sorted Set 中的所有在 `<min>` 和 `<max>` 之间的元素。

如果 Sorted Set 中的元素具有不同的分数，则返回的元素是未指定的。

使用 `memcmp()` C 函数对每个字节比较时，元素被认为是从低祷告的字符串排序。如果公共部分相同，则较长的字符串被认为大于较短的字符串。

根据 Redis 6.2.0，此命令被视为已弃用。请在新代码中使用带有 `BYLEX` 参数的 [ZRANGE](ZRANGE.md) 命令。

可选的 `LIMIT` 参数可用于仅获取匹配元素的范围（类似于 SQL 中的 SELECT LIMIT offset,count）。负的 `<count>` 返回 `<offset>` 中的所有元素。请记住，如果 `<offset>` 很大，则需要为 `<offset>` 元素遍历已排序的集合，然后才能找到要返回的元素，这可能会增加 O(N) 时间复杂度。

---

### How to specify intervals

有效的 `<min>` 和 `<max>` 必须以 `(` 或 `[` 开头，以分别指定范围区间是不包含还是包含。

*开始* 和 *结束* 为 `+` 或 `-` 的特殊值分别表示正或负无穷字符串，因此例如命令 **ZRANGEBYLEX myzset - +** 保证返回 Sorted Set 中的所有元素，前提是所有元素都具有相同的分数。

---

### Details on strings comparison

字符串作为二进制字节数组进行比较。由于 ASCII 字符集的指定方式，这意味着通常这也具有以明显的字典方式比较普通的 ASCII 字符的效果。但是如果使用非纯 ASCII 字符串（例如 utf8 字符串），则情况并非如此。

但是，用户可以对编码字符串应用转换，以便插入到 Sorted Set 中的元素的第一部分将按照用户对特定应用程序的要求进行比较。例如，如果我想添加会以不区分大小写的方式进行比较的字符串，但查询时仍然想检索真实的大小写，我可以通过以下方式添加字符串：

```
ZADD autocomplete 0 foo:Foo 0 bar:BAR 0 zap:zap
```

由于每个元素中的第一个标准化部分（在冒号字符之前），我们强制进行给定的比较。但是，在使用 [ZRANGEBYLEX](ZRANGEBYLEX.md) 查询范围后，应用程序可以像用户显示字符串的第二部分，在冒号之后。

比较的二进制特性允许使用 Sorted Set 作为通用索引，例如，元素的第一部分可以是 64 位大端数。由于大端数字在初始位置具有最高有效字节，因此而仅仅比较将使用数字进行比较。这可用于对 64 位值试试范围查询。如下例所示，在前 8 个字节之后，我们可以存储我们正在索引的元素的值。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：指定分数范围内的元素列表。

---

### Examples

```
redis> ZADD myzset 0 a 0 b 0 c 0 d 0 e 0 f 0 g
(integer) 7
redis> ZRANGEBYLEX myzset - [c
1) "a"
2) "b"
3) "c"
redis> ZRANGEBYLEX myzset - (c
1) "a"
2) "b"
redis> ZRANGEBYLEX myzset [aaa (g
1) "b"
2) "c"
3) "d"
4) "e"
5) "f"
redis> 
```