## LPUSH key element [element ...]

    起始版本：1.0.0.
    时间复杂度：对于添加的每个元素为 O(1)，因此当使用多个参数调用命令时，O(N) 添加 N 个元素。

将所有指定的值插入存储在 key 的列表的头部。如果 key 不存在，则在执行推送操作之前将其创建为空列表。当 key 保存的值不是列表时，将返回错误。

可以使用单个命令调用推送多个元素，只需在命令末尾指定多个参数。元素一个接一个地插入到列表的头部，从最左边的元素到最右边的元素。因此，例如命令 `LPUSH mylist a b c` 将生成一个包含 c 作为第一个元素、b 作为第二个元素和 a 作为第三个元素的列表。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) ：推送操作后列表的长度。

---

### History

- &gt;= 2.4：接受多个元素参数。在 2.4 之前的 Redis 版本中，可以为每个命令推送一个值。

---

### Examples

```
redis> LPUSH mylist "world"
(integer) 1
redis> LPUSH mylist "hello"
(integer) 2
redis> LRANGE mylist 0 -1
1) "hello"
2) "world"
redis> 
```