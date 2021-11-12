## GETRANGE key start end

    起始版本：2.4.0.
    时间复杂度：O(N)，其中 N 是字符串长度，复杂度由最终返回长度决定，但由于通过一个字符串创建子字符串是很容易的，它可以被认为是 O(1)。

**警告**：这个命令是被重命名为 [GETRANGE](GETRANGE.md) ，在小于 2.0 的 Redis 版本叫做 `SUBSTR`。

返回 key 对应的字符串 value 的子串，这个子串是由 start 和 end 位移决定的（两者都是包含）。可以用负的位移来表示从 string 尾部开始数的下标。所以 -1 就是最后一个字符，-2 就是倒数第二个，以此类推。

这个函数处理超出范围的请求时，都把结果限制在 string 内。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)

---

### Examples

```
redis> SET mykey "This is a string"
"OK"
redis> GETRANGE mykey 0 3
"This"
redis> GETRANGE mykey -3 -1
"ing"
redis> GETRANGE mykey 0 -1
"This is a string"
redis> GETRANGE mykey 10 100
"string"
redis> 
```