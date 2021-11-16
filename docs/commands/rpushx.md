## RPUSHX key element [element ...]

    起始版本：2.2.0。
    时间复杂度：对于添加每个元素，O(1)。因此当使用多个参数命令调用时，添加 N 个元素为 O(N)。

当且仅当 key 存在并且是一个列表，将值 value 插入到指定 key 的列表的尾部。和 [RPUSH](rpush.md) 命令相反，当 key 不存在时，该命令什么都不做。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 命令执行 push 后，列表的长度。

---

### History

- &gt;= 4.0：接受多个元素参数。在 4.0 之前的 Redis 版本中，每个命令仅能推送一个值。

---

### Examples

```
redis> RPUSH mylist "Hello"
(integer) 1
redis> RPUSHX mylist "World"
(integer) 2
redis> RPUSHX myotherlist "World"
(integer) 0
redis> LRANGE mylist 0 -1
1) "Hello"
2) "World"
redis> LRANGE myotherlist 0 -1
(empty list or set)
redis> 
```