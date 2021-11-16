## RPUSH key element [element ...]

    起始版本：1.0.0
    时间复杂度：对于添加每个元素，O(1)。因此当使用多个参数命令调用时，添加 N 个元素为 O(N)。

将所有指定的 values 插入存储在 key 的列表的尾部。如果 key 不存在，则在执行 push 操作之前将其创建为空列表。当 key 保存的值不是列表时，将返回错误。

可以使用单个命令调用 push 多个元素，只需在命令末尾指定多个参数。元素是从左到右一个接一个从列表尾部插入。比如命令 `RPUSH mylist a b c` 将会生成一个列表，其中 a 是第一个元素， b 是第二个元素，c是第三个元素。

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 推送操作后列表的长度。

### History

- &gt;= 接受多个元素参数。在 2.4 之前的 Redis 版本中，每个命令只能 push 一个值。

### Examples

```
redis> RPUSH mylist "hello"
(integer) 1
redis> RPUSH mylist "world"
(integer) 2
redis> LRANGE mylist 0 -1
1) "hello"
2) "world"
redis> 
```