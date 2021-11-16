## LINSERT key BEFORE|AFTER pivot element

    起始版本：2.0.0.
    时间复杂度：O(N)，其中 N 是在看到 value pivot 之前要遍历的元素个数。这意味着插入列表的左端(头)可以被认为是 O(1)，插入列表的右端（尾）是 O(N)。

在参考值 `pivot` 之前或之后在 key 对应的列表中插入元素。

当 key 不存在时，认为是空列表，不进行任何操作。

当 key 存在，但不是一个 list 的时候，返回 error。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：插入操作后列表的长度，或者当没有找到 value pivot 时为 -1。

---

### Examples

```
redis> RPUSH mylist "Hello"
(integer) 1
redis> RPUSH mylist "World"
(integer) 2
redis> LINSERT mylist BEFORE "World" "There"
(integer) 3
redis> LRANGE mylist 0 -1
1) "Hello"
2) "There"
3) "World"
redis> 
```