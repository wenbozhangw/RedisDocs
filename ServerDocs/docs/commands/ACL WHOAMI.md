## ACL WHOAMI

    自 6.0.0 起可用。
    时间复杂度: O(1)。

返回当前连接进行身份验证的用户名。新连接使用 "default" 用户进行身份验证。他们可以使用 [AUTH](AUTH.md) 更改用户。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 当前连接的用户名。

---

### Examples

```
> ACL WHOAMI
"default"
```