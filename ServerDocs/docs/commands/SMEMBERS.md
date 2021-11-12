## SMEMBERS key

    起始版本：1.0.0。
    时间复杂度：O(N)，其中 N 为 set 基数(cardinality)。

返回存储在 key 中的 set value 的所有成员。

该命令的作用域使用一个参数的 [SINTER](SINTER.md) 命令作用相同。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) : set 中的所有元素。

---

### Examples

```
redis> SADD myset "Hello"
(integer) 1
redis> SADD myset "World"
(integer) 1
redis> SMEMBERS myset
1) "Hello"
2) "World"
redis> 
```