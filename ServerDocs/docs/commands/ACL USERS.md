## ACL USERS

    自 6.0.0 起可用。
    时间复杂度: 0(N)。其中 N 是配置的用户数。

该命令显示 Redis ACL 系统中当前配置用户的所有用户名列表。

---

### Return Value

数组类型的字符串。

---

### Examples

```
> ACL USERS
1) "anna"
2) "antirez"
3) "default"
```