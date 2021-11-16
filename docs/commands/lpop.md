## LPOP key [count]

    起始版本：1.0.0.
    时间复杂度：O(N) 其中 N 为返回的元素数量。

删除并返回存储在 key 中的列表的第一个元素。

默认情况下，该命令从列表的开头弹出单个元素。当提供可选的 `count` 参数时，返回将最多包含 `count` 个元素，具体取决于列表的长度。

---

### Return Value

当不使用 count 参数调用：

 [Bulk string reply](../topics/protocol.md#resp-bulk-strings) ：第一个元素的值，或如果 key 不存在，返回 nil。
 
当使用 count 参数调用：

[Array reply](../topics/protocol.md#resp-arrays)：弹出的元素列表，或如果 key 不存在，返回 nil。

---

### History

- &gt;= 6.2：添加了 `count` 参数。

---

### Examples

```
redis> RPUSH mylist "one" "two" "three" "four" "five"
(integer) 5
redis> LPOP mylist
"one"
redis> LPOP mylist 2
1) "two"
2) "three"
redis> LRANGE mylist 0 -1
1) "four"
2) "five"
redis> 
```