## RPOP key [count]

    自 1.0.0 起可用。
    时间复杂度：O(N)，其中 N 是返回的元素数。

删除并返回存在 key 的 list 的最后一个元素。

默认情况下，该命令从 list 末尾弹出单个元素。党提供可选的参数 `count` 时，将会返回最多包含 count 个元素，具体取决于 list 的长度。

### Return Value

在不带 count 参数的情况下调用时：

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 最后一个元素的值，当 key 不存在时为 nil。

当使用 count 参数调用时：

[Array reply](../topics/protocol.md#resp-arrays) : 弹出元素的列表，当 key 不存在时为 nil。

### History

- &gt;= 6.2 ： 添加了 count 参数。

### Examples

```
redis> RPUSH mylist "one" "two" "three" "four" "five"
(integer) 5
redis> RPOP mylist
"five"
redis> RPOP mylist 2
1) "four"
2) "three"
redis> LRANGE mylist 0 -1
1) "one"
2) "two"
redis> 
```