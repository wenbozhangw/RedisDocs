## ACL LIST

    自 6.0.0 起可用。
    时间复杂度: 0(N)。其中 N 是配置的用户数。

该命令显示 Redis 服务器中当前激活的 ACL 规则。返回数组中的每一行定义了一个不同的用户，格式与 redis.conf 文件或外部 ACL 文件中使用的格式相同，因此您可以将 ACL LIST 命令返回的内中直接剪切并粘贴到配置文件中，如果您愿意（但一定要检查 [ACL SAVE](ACL SAVE.md) ）。

---

### Return Value

数组类型的字符串。

---

### Examples

```
> ACL LIST
1) "user antirez on #9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08 ~objects:* &* +@all -@admin -@dangerous"
2) "user default on nopass ~* &* +@all"
```