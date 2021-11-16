## SWAPDB index1 index2

    起始版本：4.0.0。
    时间复杂度：O(N)，其中 N 为 watching 或 blocking 来自两个数据库的 key 的客户端数量。

此命令交换两个 Redis 数据库，以便连接到给定数据库的所有客户端立即看到另一个数据库的数据，反之亦然。例子：

```
SWAPDB 0 1
```

这会将数据库 0 与数据库 1 交换。所有与数据库 0 连接的客户端将立即看到新数据，就像所有与数据库 1 连接的客户端将看到以前来自数据库 0 的数据一样。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings): 如果 [SWAPDB](swapdb.md) 执行正确，则返回 OK。

---

### Examples

```
SWAPDB 0 1
```