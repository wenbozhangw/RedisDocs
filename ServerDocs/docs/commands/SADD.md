## SADD key member [member ...]

    起始版本：1.0.0.
    时间复杂度：对于每添加一个元素为 O(1)，因此当使用多个参数调用命令时，添加 N 个元素为 O(N)。

添加一个或多个指定的 member 元素到 key 对应的集合中。已经是此集合的指定成员将被忽略。如果 key 不存在，则在添加指定成员之前创建一个新集合。

当 key 中存储的值不是集合时，会返回错误。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：添加到集合中的元素数，不包括集合中已经存在的所有元素。

---

### History

- &gt;= 2.4：接受多个成员参数。Redis 2.4 之前的版本每次调用只能添加一个成员。

---

### Examples

```
redis> SADD myset "Hello"
(integer) 1
redis> SADD myset "World"
(integer) 1
redis> SADD myset "World"
(integer) 0
redis> SMEMBERS myset
1) "World"
2) "Hello"
redis> 
```