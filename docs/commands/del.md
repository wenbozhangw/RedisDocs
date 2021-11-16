## DEL key [key ...]

    起始版本：1.0.0
    时间复杂度：O(N)，其中 N 为将要被删除的 key 的数量。当删除的 key 是字符串以外的复杂数据类型时，此 key 的时间复杂度为 O(M)，比如 List, Set, Hash。删除单个字符串 key 的时间复杂度是 O(1)。

删除指定的一批 keys，如果删除的某些 key 不存在，则直接忽略。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) ：被删除的keys的数量。

---

### Examples

```
redis> SET key1 "Hello"
"OK"
redis> SET key2 "World"
"OK"
redis> DEL key1 key2 key3
(integer) 2
redis> 
```