## SPOP key [count]

    自 1.0.0 起可用。
    时间复杂度：没有 count 参数为 O(1)，否则为 O(N)，其中 N 是传递的 count 值。

从存储在 key 的集合中移除并返回一个或多个随机元素。

此操作与 [SRANDMEMBER](srandmember.md) 类似，它从一个集合中返回一个或多个随机元素，但不删除元素。

默认情况下，该命令从集合中弹出一个成员。当提供可选的 count 参数时，恢复将最多包含 count 个成员，具体取决于集合的元素数量。

### Return Value

在不带 count 参数的情况下调用时：

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 删除的成员，或者当 key 不存在时 nil。

当使用 count 参数调用时：

[Array reply](../topics/protocol.md#resp-arrays) : 删除的成员，或者当 key 不存在时为空数组。

### History

- &gt;= 3.2 : 增加 count 参数。

### Examples

```
redis> SADD myset "one"
(integer) 1
redis> SADD myset "two"
(integer) 1
redis> SADD myset "three"
(integer) 1
redis> SPOP myset
"three"
redis> SMEMBERS myset
1) "two"
2) "one"
redis> SADD myset "four"
(integer) 1
redis> SADD myset "five"
(integer) 1
redis> SPOP myset 3
1) "two"
2) "one"
3) "five"
redis> SMEMBERS myset
1) "four"
redis> 
```

### Distribution of returned elements

请注意，当您需要保证返回元素的均匀分布时，次领了不适用。更多有关 [SPOP](spop.md) 使用的算反信息，请查阅 Knuth 采样和 Floyd 采样算法。