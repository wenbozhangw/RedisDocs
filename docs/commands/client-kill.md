## CLIENT KILL [ip:port] [ID client-id] [TYPE normal|master|slave|pubsub] [USER username] [ADDR ip:port] [LADDR ip:port] [SKIPME yes/no]

    起始版本：2.4.0.
    时间复杂度：O(N)，其中 N 为客户端连接数。

[CLIENT KILL](client-kill.md) 命令关闭一个指定的连接。此命令支持两种格式，旧的格式为：

```
CLIENT KILL addr:port
```

`ip:port` 应该是 [CLIENT LIST](client-list.md) 命令里面列出的客户端连接之一。

新的格式为：

```
CLIENT KILL <filter> <value> ... ... <filter> <value>
```
新的格式可以根据不同属性杀死客户端而不是只按照地址杀死。它们有以下一些格式：
- `CLIENT KILL ADDR ip:port`。和旧版的三个参数时的行为完全一样。
- `CLIENT KILL LADDR ip:port`。杀死所有连接到指定 local (bind) address 的客户端。
- `CLIENT KILL ID client-id`。允许通过其唯一 ID 字段杀死客户端。使用 [CLIENT LIST](client-list.md) 命令检索客户端 ID。
- `CLIENT KILL TYPE type`，其中 _type_ 可以是 `normal、master、replica、pubsub` 之一。这将关闭所有特殊类的客户端。请注意，被认为是属于 `normal` 类的客户端将会被 [MONITOR](monitor.md) 命苦监听到。
- `CLIENT KILL USER username`。关闭使用指定 [ACL](../topics/acl.md) 用户名进行身份验证的所有连接，但如果用户名未对应到现有的 ACL 用户，则返回错误。
- `CLIENT KILL SKIPME yes/no`。默认情况下，该选项设置为 yes，即不会杀死调用命令的客户端，但将此选项设置为 no 会同时杀死调用命令的客户端。

可以在执行命令的同时给定多个筛选条件。该命令会使用逻辑 AND 来处理多个筛选条件，比如：

```
CLIENT KILL addr 127.0.0.1:12345 type pubsub
```

是可以的，并且只会杀死给定 IP 地址的 `pubsub` 客户端。这种包含多个过滤器的格式目前很少使用。

当使用新格式时，该命令不再返回 OK 或者错误，而是返回被杀死的客户端个数，有可能为 0。

---

### CLIENT KILL and Redis Sentinel

当前版本的 Redis Sentinel 可以在实例被重新配置的时候使用 CLIENT KILL 杀死客户端。这样可以强制客户端和一个 Sentinel 重新连接并更新自己的配置。

---

### Notes

因为 Redis 的单线程属性，不可能在客户端执行命令时杀掉它。从客户端的角度看，永远无法杀死一个正在执行命令的连接。但是，当客户端发送下一条命令时会意识到连接已被关闭，原因为网络错误。

---

### Return Value

三个参数格式执行命令：

[Simple string reply](../topics/protocol.md#resp-simple-strings) ：连接存在并被关闭返回 OK。

使用 filter/value 格式执行命令：

[Integer reply](../topics/protocol.md#resp-integers)：被杀死的客户端个数。

---

### History

- &gt;= 2.8.12：增加新地过滤格式。
- &gt;= 2.8.12：ID 选项。
- &gt;= 3.2：为 [TYPE](type.md) 选项增加 `master` 类型。
- &gt;= 5：将 `slave` [TYPE](type.md) 替换为 `replica`。`slave` 仍然支持向后兼容。
- &gt;= 6.2：`LADDR` 选项。