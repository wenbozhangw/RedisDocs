## ACL GETUSER username

    自 6.0.0 起可用。
    时间复杂度: 0(N)。其中 N 是用户拥有的密码 (password)、命令 (command) 和 模式 (pattern) 规则的数量。

该命令返回为现有 ACL 用户定义的所有规则。具体来说，它列出了用户的 ACL flags、password hashes and key name patterns。请注意，command rules 作为字符串返回，格式与 [ACL SETUSER](acl-setuser.md) 命令使用的格式相同。 command rules 的描述反映了用户的有效权限，因此虽然它可能与用于配置用户的规则集不同，但在功能上仍然相同。

---

### Return Value

[Array reply](../topics/protocol.md#RESP Arrays) : 用户的 ACL 规则定义列表。

---

### History

- &gt;= 6.2 : 添加了 Pub/Sub channel patterns。

---

### Examples

这是默认用户的默认配置：

```
> ACL GETUSER default
1) "flags"
2) 1) "on"
   2) "allkeys"
   3) "allcommands"
   4) "nopass"
3) "passwords"
4) (empty array)
5) "commands"
6) "+@all"
7) "keys"
8) 1) "*"
9) "channels"
10) 1) "*"
```