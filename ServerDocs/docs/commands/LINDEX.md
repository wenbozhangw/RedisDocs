## LINDEX key index

    起始版本：1.0.0.
    时间复杂度：O(N) 其中 N 是要遍历以到达索引处的元素的元素数量。这使得请求列表第一个或最后一个元素为 O(1)。

返回存储在 key 对应的列表中索引 `index` 处的元素。索引从零开始，因此 0 表示第一个元素，1 表示第二个元素，以此类推。负索引可用于指定从列表尾部开始的元素。这里，-1 表示最后一个元素，-2 表示倒数第二个，以此类推。

当 key 对应的值不是列表时，返回错误。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)：请求的元素，当索引超出范围时为 nil。

---

### Examples

```
redis> LPUSH mylist "World"
(integer) 1
redis> LPUSH mylist "Hello"
(integer) 2
redis> LINDEX mylist 0
"Hello"
redis> LINDEX mylist -1
"World"
redis> LINDEX mylist 3
(nil)
redis> 
```