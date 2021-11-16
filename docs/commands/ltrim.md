## LTRIM key start stop

    起始版本：1.0.0.
    时间复杂度：O(N) 其中 N 为操作要删除的元素数量。

修剪现有列表，使其仅包含指定范围的指定元素。`start` 和 `stop` 都是从零开始的索引，其中 0 是列表的第一个元素（头部），1 是下一个元素，以此类推。

例如：`LTRIM foobar 0 2` 将修改存储在 foobar 中的列表，以便仅保留列表的前三个元素。

`start` 和 `end` 也可以是负数，表示与列表末尾的偏移量，比如 -1 表示列表里的最后一个元素，-2 表示倒数第二个，以此类推。

超过范围的并不会产生错误：如果 start 超过列表尾部，或 start > end，结果会是列表变成空表（即该 key 会被移除）。如果 end 超过列表尾部，Redis 会将其作为列表的最后一个元素。

[LTRIM](ltrim.md) 的一个常见用法是和 [LPUSH](lpush.md)/[RPUSH](rpush.md) 一起使用。例如：

```
LPUSH mylist someelement
LTRIM mylist 0 99
```

这一对命令会将一个新的元素 push 进列表里，并报在该列表不会增长超过 100 个元素。比如当用 Redis 存储日志。需要特别注意的是，当用这种方式来使用 [LTRIM](ltrim.md) 的时候，操作的复杂度是 O(1)，因为平均情况下，每次只会有一个元素会被移除。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)

---

### Examples

```
redis> RPUSH mylist "one"
(integer) 1
redis> RPUSH mylist "two"
(integer) 2
redis> RPUSH mylist "three"
(integer) 3
redis> LTRIM mylist 1 -1
"OK"
redis> LRANGE mylist 0 -1
1) "two"
2) "three"
redis> 
```