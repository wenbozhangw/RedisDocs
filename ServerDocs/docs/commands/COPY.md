## COPY source destination [DB destination-db] [REPLACE]

    起始版本：6.2.0.
    时间复杂度：最坏情况集合为 O(N)，其中 N 是嵌套项的数量。字符串值为 O(1)。

此命令将存储在 `source` key 中的 value 复制到 `destination` key 中。

默认情况下，`destination` key 是在连接使用的逻辑数据库中创建的。`DB` 选项允许为 `destination` key 指定替代逻辑数据库索引。

当 `destination` key 已经存在时，该命令会返回错误。`REPLACE` 选项可以在复制前删除该键，之后再将值复制到 `destination` key 。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- 如果源复制完成，返回 1.
- 如果源没有复制，返回 0.

---

### Examples

```
SET dolly "sheep"
COPY dolly clone
GET clone
```