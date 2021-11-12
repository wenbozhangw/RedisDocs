## HELLO [protover [AUTH username password] [SETNAME clientname]]

    起始版本：6.0.0。
    时间复杂度：O(1)。

切换到不同的协议，可选择验证和设置连接名称，或提供上下文客户端报告。

Redis 6 及以上版本支持两种协议：旧协议 RESP2 和 Redis 6 引入的新协议 RESP3。RESP3 有一定的优势，因为当连接处于这种模式时，Redis 能够回复更多语义回复(semantical replies)：例如，[HGETALL](HGETALL.md) 将返回一个 map 类型，因此客户端库实现不再需要在将数据返回给调用者之前预先知道如何将其转换为 hash。要全力了解 RESP3，请 [check this repository](https://github.com/antirez/resp3) 。

在 Redis 6 中，连接以 RESP2 模式启动，因此实现 RESP2 的客户端不需要更新或更改。尽管未来版本可能会默认支持 RESP3，但短期内没有取消对 RESP2 支持的计划。

[HELLO](HELLO.md) 总是回复当前服务器的连接属性的列表，例如：versions、modules loaded、client ID、replication role 等。在 Redis 6.2 中不带有任何参数调用并且默认使用 RESP2 协议时，回复如下所示：

```
> HELLO
 1) "server"
 2) "redis"
 3) "version"
 4) "255.255.255"
 5) "proto"
 6) (integer) 2
 7) "id"
 8) (integer) 5
 9) "mode"
10) "standalone"
11) "role"
12) "master"
13) "modules"
14) (empty array)
```

想要使用 RESP3 模式握手的客户端需要调用 [HELLO](HELLO.md) 命令并指定值 "3" 作为 `protover` 参数，如下所示：

```
> HELLO 3
1# "server" => "redis"
2# "version" => "6.0.0"
3# "proto" => (integer) 3
4# "id" => (integer) 10
5# "mode" => "standalone"
6# "role" => "master"
7# "modules" => (empty array)
```

因为 [HELLO](HELLO.md) 回复了有用的信息，并且考虑到 `protover` 是可选的或可以设置为 "2"，客户端库作者可能会在设置连接时考虑使用此命令而不是规范的 [PING](ping.md)。

当使用可选的 `protover` 参数调用时，此命令将协议切换到指定版本，并接受以下选项：
- `AUTH <username> <password>` ：除了切换到指定的协议版本外，还可以直接验证连接。这使得在建立新连接时不需要在 [HELLO](HELLO.md) 之前调用 [AUTH](AUTH.md)。请注意，用户名可以设置为 "default" 已针对不使用 ACL 的服务器进行身份验证，而是使用 Redis 6 版本之前的更简单的 `requirepass` 机制。
- SETNAME <clientname> ：这相当于调用 [CLIENT SETNAME](CLIENT SETNAME.md) 。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) : 服务器属性列表。回复是选中 RESP3 时的 map 而不是数组。如果请求的 `protover` 不存在，该命令将返回错误。

---

### History

- &gt;= : `protover` 可选；当不带参数调用时，该命令报告当前连接的上下文。
