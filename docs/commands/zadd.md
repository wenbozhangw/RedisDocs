## ZADD key [NX|XX] [GT|LT] [CH] [INCR] score member [score member ...]

    起始版本：1.2.0.
    时间复杂度：O(log(N)) 对于添加的每个项目，其中 N 是 Sorted Set 中的元素数。

将具有指定分数的所有指定成员添加到存储在 key 的 Sorted Set 中。可以指定多个分数/成员(score/member)对。如果指定的成员已经是 Sorted Set 的成员，则更新分数并在正确的位置重新插入元素以确保正确排序。

如果 key 不存在，则会创建一个新的 Sorted Set，将指定成员作为唯一成员，就像 Sorted Set 为空一样。如果 key 存在但不是 Sorted Set，则返回错误。

分数值应该是双精度浮点数的字符串表示形式。 `+inf` 和 `-inf` 值也是有效值。

---

### ZADD options

ZADD 支持选项列表，在 key 名之后和第一个 score 参数之前指定。选项是：
- **XX** ：只更新已经存在的元素。不要添加新元素。
- **NX** ：只添加新元素。不要更新已经存在的元素。
- **LT** ：如果新分数小于当前分数，则仅更新现有元素。此标志不会阻止添加新元素。
- **GT** ：如果新分数大于当前分数，则仅更新现有元素。此标志不会阻止添加新元素。
- **CH** ：将返回从添加的新元素数修改为更改的元素总数（CH 是 *changed* 的缩写）。更改的元素是添加的新元素和已更新分数的现有元素。因此，命令行中指定的具有与过去相同分数的元素不计算在内。注意：通常 [ZADD](zadd.md) 的返回值只计算添加的新元素的数量。
- **INCR** ：指定此选项时，[ZADD](zadd.md) 的作用类似于 [ZINCRBY](zincrby.md) 。在这种模式下只能指定一个分数元素对。

注意：**GT**、**LT** 和 **NX** 选项是互斥的。

---

### Range of integer scores that can be expressed precisely

Redis Sorted Set 使用一个 _double 64-bit floating number_ 来表示分数。在我们支持的所有架构中，这表示为 **IEEE 754 floating point number**，能够精确表示包括 `-(2^53)` 和 `+(2^53)` 之间的整数。在更实际的术语中，-9007199254740992 和 9007199254740992 之间的所有整数都可以完美表示。更大的整数在内部使用指数形式表示，所以，如果为分数设置一个非常大的整数，你得到的是一个近似的十进制数。

---

### Sorted sets 101

Sorted sets 按照分数以递增的方式进行排序。相同的成员(member)只存在一次，Sorted set 不允许存在重复的成员。分数可以通过 [ZADD](zadd.md) 命令进行更新或者也可以通过 [ZINCRBY](zincrby.md) 命令递增来修改之前的值，相应地他们的排序位置也会随着分数变化而变化。

获取一个成员当前的分数可以使用 [ZSCORE](zscore.md) 命令，也可以用它来验证成员是否存在。

更多原语 Sorted sets 的信息请参考 [sorted sets](../topics/data-types.md#sorted-sets) 。

---

### Elements with the same score

Sorted sets 里面的成员是不能重复的，都是唯一的，但是，不同的成员间可能有 _相同的分数_ 。当多个成员有相同的分数时，它们将 _ordered lexicographically (有序字典排序)_ （仍由分数作为第一排序条件，然后，相同分数的成员按照字典规则相对排序）。

字典顺序排序用的是二进制，它比较的是字符串的字节数组。

如果用户将所有元素设置相同分数（例如0），Sorted sets 中的所有元素将按照字典顺序进行排序，范围查询元素可以使用 [ZRANGEBYLEX](zrangebylex.md) 命令（注：范围查询分数可以使用 [ZRANGEBYSCORE](zrangebyscore.md) 命令）。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：

- 在不带可选参数的情况下使用时，添加到 Sorted sets 中的元素数量（不包括更新分数的元素）。
- 如果指定的 `CH` 选项，则返回更改的元素数（添加或更新）。

如果指定了 [INCR](incr.md) 选项，则返回值将是 [Bulk string reply](../topics/protocol.md#resp-bulk-strings)：

- 成员(member) 的新分数（双精度浮点数）表示为字符串，如果操作被终止（使用 `XX` 或 `NX` 选项调用时），则为 nil。

---

### History

- &gt;= 2.4：接收多个 member。在 Redis 2.4 之前，命令只能添加或更新一个 member。
- &gt;= 3.0.2：添加 `XX`、`NX`、`CH` 和 [INCR](incr.md) 选项。
- &gt;= 6.2：添加 `GT` 和 `LT` 选项。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 1 "uno"
(integer) 1
redis> ZADD myzset 2 "two" 3 "three"
(integer) 2
redis> ZRANGE myzset 0 -1 WITHSCORES
1) "one"
2) "1"
3) "uno"
4) "1"
5) "two"
6) "2"
7) "three"
8) "3"
redis> 
```