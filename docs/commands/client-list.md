## CLIENT LIST [TYPE normal|master|replica|pubsub] [ID client-id [client-id ...]]

    起始版本：2.4.0.
    时间复杂度：O(N)，其中 N 为客户端连接的数量。

[CLIENT LIST](client-list.md) 命令以大多数人类可读格式返回有关客户端连接服务器的信息和统计信息。

您可以使用可选子命令之一来过滤列表。`TYPE type` 子命令按客户端的类型过滤列表，其中 `type` 是 `normal`、`master`、`replica` 和 `pubsub` 之一。请注意，被 [MONITOR](monitor.md) 命令阻塞的客户端属于 `normal`。

`ID` 过滤器仅返回 ID 与 client-id 参数匹配的客户端的条目。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 一个唯一的字符串，格式如下：
- 每行一个客户端连接（用 LF 分隔）
- 每行由一系列由空格字符串分割的 property=value 字段组成。

以下是字段的含义：
- `id` : 一个唯一 64 位客户端 ID。
- `name` : 客户端使用 [CLIENT SETNAME](client-setname.md) 设置的名称。
- `addr` : 客户端的 address/port。
- `laddr` : local address 客户端连接的 address/port (bind address)。
- `fd` : socket 对应的文件描述符(file descriptor)。
- `age` : 连接的总持续时间（以秒为单位）。
- `idle` : 连接的空闲时间（以秒为单位）。
- `flags` : client flags（见下文）。
- `db` : 当前数据库 ID。
- `sub` : channel 的订阅数量。
- `psub` : pattern matching 订阅数量。
- `multi` : MULTI/EXEC 上下文中的命令数量。
- `qbuf` : 查询缓冲区长度（0 表示没有待处理的查询）。
- `qbuf-free` : 查询缓冲区的可用空间（0 表示缓冲区已满）。
- `obl` : 输出缓冲区长度。
- `oll` : 输出列表长度（当缓冲区已满时，回复在此列表中排队）。
- `omem` : 输出缓冲区内存使用。
- `events` : 文件描述符事件(file descriptor events)（见下文）。
- `cmd` : played 最后一个命令。
- `argv-mem` : 下一个命令的不完整参数（已从查询缓冲区中取出）。
- `tot-mem` : 此客户端在其各种缓冲区中消耗的总内存。
- `redir` : 当前客户端跟踪重定向的客户端 id。
- `user` : 经过身份验证的客户端用户名。

client flags 可以是以下各项的组合：

```
A: 尽可能快地关闭连接
b: 客户端正在等待阻塞事件
c: 在将回复完整地写出之后，关闭链接
d: 一个受监视（watched）的键已被修改 - EXEC 命令将失败
i: 客户端正在等待 VM I/O 操作（已废弃）
M: 客户端是主节点（master）
N: 未设置任何 flag
O: 客户端是 MONITOR 模式下的附属节点（slave）
P: 客户端是 Pub/Sub 订阅者
r: 客户端是只读模式的集群节点
S: 客户端是一般模式下（normal）的附属节点
u: 客户端未被阻塞（unblocked）
U: 通过Unix套接字连接的客户端
x: 客户端正在执行 MULTI/EXEC 上下文
t: 客户端启用 key 跟踪以执行客户端缓存
R: 客户跟踪目标客户端无效
B: 客户端启用广播跟踪模式
```

文件描述符事件可以是：

```
r: 客户端套接字（在事件 loop 中）是可读的（readable）
w: 客户端套接字（在事件 loop 中）是可写的（writeable）
```

---

### Notes

新字段会随着测试有规律的天街。某些字段将来可能会被删除。一个版本安全的 Redis 客户端使用这个命令时应该根据字段解析相应的内容。（比如：处理位置的字段，应跳过该字段）。

---

### History

- &gt;= 2.8.12 : 增加唯一的 client id 字段。
- &gt;= 5.0 : 增加可选 [TYPE](type.md) 过滤器。
- &gt;= 6.2 : 增加了 `laddr` 字段和可选的 `ID` 过滤器。