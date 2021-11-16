## LPUSHX key element [element ...]

    起始版本：2.2.0.
    时间复杂度：对于添加的每个元素 O(1)，因此当使用多个参数调用，添加 N 个元素时 O(N)。

仅当 key 已经存在并持有一个列表时，才在 key 对应的存储的列表的头部插入指定的值。与 [LPUSH](lpush.md) 相反，当 key 不存在时，不会执行任何操作。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：在 push 操作后的 list 长度。

---

### History

- &gt;= 4.0：接收多个元素参数。在 4.0 之前的 Redis 版本中，可以为每个命令推送一个值。

---

### Examples

```
redis> LPUSH mylist "World"
(integer) 1
redis> LPUSHX mylist "Hello"
(integer) 2
redis> LPUSHX myotherlist "Hello"
(integer) 0
redis> LRANGE mylist 0 -1
1) "Hello"
2) "World"
redis> LRANGE myotherlist 0 -1
(empty list or set)
redis> 
```