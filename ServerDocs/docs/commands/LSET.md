## LSET key index element

    起始版本：1.0.0
    时间复杂度：O(N) 其中 N 是列表的长度。设置列表的第一个或最后一个元素是 O(1)。

将索引处的列表元素设置为 `element`。有关 `index` 参数的更多信息，请参阅 [LINDEX](LINDEX.md)。

超出范围的索引会返回错误。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)

---

### Examples

```
redis> RPUSH mylist "one"
(integer) 1
redis> RPUSH mylist "two"
(integer) 2
redis> RPUSH mylist "three"
(integer) 3
redis> LSET mylist 0 "four"
"OK"
redis> LSET mylist -2 "five"
"OK"
redis> LRANGE mylist 0 -1
1) "four"
2) "five"
3) "three"
redis> 
```