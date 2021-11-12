## CLIENT ID

    自 5.0.0 起可用。
    时间复杂度：O(1)。

该命令仅返回当前连接的 ID。每个连接 ID 都有一定地保证：
1. 它永远不会重复，所以如果 [CLIENT ID](CLIENT ID.md) 返回相同的数字，调用者可以确定底层客户端没有端口并重新连接 connection，但它仍然是相同地连接。 
2. ID 是单调递增的。如果一个连接的 ID 大于另一个连接的 ID，则保证第二个与服务器建立连接较晚。

此命令与 [CLIENT UNBLOCK](CLIENT UNBLOCK.md) 一起使用特别有用，[CLIENT UNBLOCK](CLIENT UNBLOCK.md) 也在 Redis 5 中与 [CLIENT ID](CLIENT ID.md) 一起引入。检查 [CLIENT UNBLOCK](CLIENT UNBLOCK.md) 命令页面以获取涉及这两个命令的模式。

---

### Examples

```
redis> CLIENT ID
ERR Unknown or disabled command 'CLIENT'
redis> 
```

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) ：客户端的 ID。
