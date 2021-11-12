## ACL SAVE

    自 6.0.0 起可用。
    时间复杂度: 0(N)。其中 N 是配置的用户数。

当 Redis 配置为使用 ACL 文件（带有 aclfile 配置选项）时，此命令会将当前定义的 ACL 从服务器内存保存到 ACL 文件中。

---

### Return Value

[Simple string reply](../topics/protocol.md#RESP Simple Strings) : OK on success.

由于多种原因，改名了可能会失败并出现错误：如果无法写入文件或服务器未配置为使用外部 ACL 文件。

---

### Examples

```
> ACL SAVE
+OK

> ACL SAVE
-ERR There was an error trying to save the ACLs. Please check the server logs for more information.
```