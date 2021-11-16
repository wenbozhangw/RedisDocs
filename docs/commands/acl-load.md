## ACL LOAD

    自 6.0.0 起可用。
    时间复杂度: 0(N)。其中 N 是配置的用户数。

当 Redis 配置为使用 ACL 文件（使用 `aclfile` 配置选项）时，此命令将从文件中重新加载 ACL，将所有当前 ACL 规则替换为文件中定义的规则。该命令确保具有全部都有 (*all*) 或全部都没有 (*nothing*) 的行为，即：

- 如果文件中的每一行都有效，则加载所有 ACL。
- 如果文件中的一行或多行无效，则不会加载任何内容，并继续使用服务器内存中定义的旧 ACL 规则。

---

### Return Value

[Simple string reply](../topics/protocol.md#RESP Simple Strings) : OK on success.

该命令可能会因以下几种原因失败并出现错误：如果文件不可读，如果文件内部有错误，在这种情况下，错误将在 error 中报告给用户。最后，如果服务器未配置为使用外部 ACL 文件，该命令将失败。

---

### Examples

```
> ACL LOAD
+OK

> ACL LOAD
-ERR /tmp/foo:1: Unknown command or category name in ACL...
```