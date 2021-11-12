## AUTH [username] password

    自 1.0.0 起可用。

AUTH 命令在两种情况下对当前连接进行身份验证：
1. 如果 Redis 服务器通过 `requirepass` 选项受密码保护。
2. 如果 Redis 6.0 或更高版本的实例使用 [Redis ACL system](../topics/ACL.md) 。

Redis 6 之前的 Redis 版本只能理解命令的单参数版本：

```
AUTH <password>
```

此表单仅针对使用 `requirepass` 设置的密码进行身份验证。在此配置中，Redis将拒绝刚刚连接的客户端执行的任何命令，除非连接通过 [AUTH]() 进行身份验证。

如果通过 AUTH 提供的密码与配置文件中的密码匹配，则服务器回复 OK 状态码并开始接受命令。否则，返回错误并且客户端需要尝试新密码。

使用 Redis ACL 时，应以扩展方式给出命令：

```
AUTH <username> <password>
```

为了使用 ACL 列表（请参阅 [ACL SETUSER](ACL SETUSER.md)）和官方 [ACL guide](../topics/ACL.md) 中定义的连接之一来验证当前连接以获取更多信息。

使用 ACL 时，命令的单参数形式（仅指定密码）界定隐式用户名是 "default"。

---

### History

- &gt;= 6.0.0 : 添加了 ACL style（username and password）。

---

### Security notice

由于 Redis 的高性能特性，可以在很短的时间内并行尝试大量密码，因此请确保生成一个强大且很长的密码，这样这种攻击是不可行的。生成强密码的一个好方法是通过 [ACL GENPASS](ACL GENPASS.md) 命令。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) , 如果密码或 用户名/密码 无效，回复错误。