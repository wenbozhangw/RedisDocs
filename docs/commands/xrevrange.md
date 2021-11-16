## XREVRANGE key end start [COUNT count]

    起始版本：5.0.0。
    时间复杂度：O(N)，其中 N 是返回的元素数量。如果 N 是常数（例如总是用 COUNT 要求前 10 个元素），你可以认为它是 O(1)。

此命令与 [XRANGE](xrange.md) 完全相同，但显著的区别是以相反的顺序返回条目，并以相反的顺序获取 start-end 参数：在 [XREVRANGE](xrevrange.md) 中，你需要先指定 *结束 ID*，再指定 *开始 ID*，该命令就会从 *结束 ID*侧开始生产两个 ID 之间（或完全相同）的所有元素。

因此，例如，要获得从较高 ID 到较低 ID 的所有元素，可以使用：

```
XREVRANGE somestream + -
```

类似于只获取添加到流中的最后一个元素，可以使用：

```
XREVRANGE somestream + - COUNT 1
```

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：

该命令返回 ID 与指定范围匹配的条目。返回的条目是完整的，这意味着 ID 和所有组成条目的字段都将返回。此外，返回的条目及其字段和值的顺序与使用 [XADD](xadd.md) 添加它们的顺序完全一致。

---

### History

- &gt;= 6.2 : 增加 exclusive 范围。

---

### Examples

```
redis> XADD writers * name Virginia surname Woolf
"1632825795889-0"
redis> XADD writers * name Jane surname Austen
"1632825795890-0"
redis> XADD writers * name Toni surname Morrison
"1632825795890-1"
redis> XADD writers * name Agatha surname Christie
"1632825795890-2"
redis> XADD writers * name Ngozi surname Adichie
"1632825795891-0"
redis> XLEN writers
(integer) 5
redis> XREVRANGE writers + - COUNT 1
1) 1) "1632825795891-0"
   2) 1) "name"
      2) "Ngozi"
      3) "surname"
      4) "Adichie"
redis> 
```