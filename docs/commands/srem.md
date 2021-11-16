## SREM key member [member ...]

    起始版本：1.0.0.
    时间复杂度：O(N) 其中 N 是被移除的成员的数量。

从存储在 key 的集合中删除指定的成员。不属于刺激和的指定成员将被忽略。如果 key 不存在，则将其视为空集合，此命令返回 0。

key 中存储的值不是集合时，会返回错误。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：从集合中删除的成员数，不包含不存在的成员。

---

### History

- &gt;= 2.4：接受多个成员参数。Redis 2.4 之前的版本每次调用只能删除一个集合成员。

---

### Examples

```
redis> SADD myset "one"
(integer) 1
redis> SADD myset "two"
(integer) 1
redis> SADD myset "three"
(integer) 1
redis> SREM myset "one"
(integer) 1
redis> SREM myset "four"
(integer) 0
redis> SMEMBERS myset
1) "three"
2) "two"
redis> 
```