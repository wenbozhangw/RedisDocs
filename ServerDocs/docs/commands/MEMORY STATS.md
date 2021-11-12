## MEMORY STATS

    起始版本：4.0.0.

命令 [MEMORY STATS](MEMORY%20STATS.md) 将服务器的内存使用情况以数组情况返回。

内存使用信息以指标和相对值的格式返回，如下指标会被返回：
- `peak.allocated`：Redis 启动以来，allocator 分配的内存峰值，单位字节（参考 [INFO](INFO.md) 的 `used_memory_peak`）。
- `total.allocated`：allocator 当前分配的内存总字节数（参考 [INFO](INFO.md) 的 `used_memory`）。
- `startup.allocated`：Redis 启动完成消耗的内存字节数（参考 [INFO](INFO.md) 的 `used_memory_startup`）。
- `replication.backlog`：Redis replication backlog 内存字节数（参考 [INFO](INFO.md) 的 `repl_backlog_active`）。
- `clients.slave`：所有副本节点内存消耗总字节数（查询输出缓冲区，连接内存消耗）
- `clients.normal`：Redis 所有常规客户端内存消耗总字节数（查询输出缓冲区，连接内存消耗）
- `aof.buffer`：当前和重写 AOF 缓冲区内存消耗总字节数（参考 [INFO](INFO.md) 的 `aof_buffer_length` 和 `aof_rewrite_buffer_length`）。
- `lua_caches`：Lua 脚本缓存开销的总大小（以字节为单位）。
- `dbXXX`：每个数据库中元数据占用的额外内存字节数。(Redis 的 db 就是一张 hash 表，首先就是这张 hash 表使用的内存，每个 key-value 对都有一个 dictEntry 来记录他们的关系，原信息便包含在该 db 中所有 dictEntry 使用的内存，Redis 使用 redisObject 来描述 value 所对应的不同数据类型(string、list、hash、set、zset)，那么 redisObject 占用的空间也计算在元信息中。 `overhead.hashtable.main` 指以上三者之和。 `overhead.hashtable.expires` 用于存储 key 的过期时间耗费的内存资源)。
- `overhead.total`：Redis 所有开销的总和，即 `startup.allocated`、`replication.backlog`、`clients.slaves`、`clients.normal`、`aof.buffer`，以及用于管理 Redis keyspace 的内部数据结构的总和（请参阅 [INFO](INFO.md) 的 `used_memory_overhead`）。
- `keys.count`：服务器中所有数据库中存储的 key 总数。
- `keys.bytes-per-key`：每个 key 平均字节数，**net memory usage**（`total.allocated` 减去 `startup.allocated`）与 `keys.count` 的比值。
- `dataset.bytes`：数据集的大小（以字节为单位），即从 `total.allocated` 中减去 `overhead.total` （参见 [INFO](INFO.md) 的 `used_memory_dataset`）。
- `dataset.percentage`：`dataset.bytes` 占经内存使用量的百分比。
- `peak.percentage`：`peak.allocated` 在 `total.allocated` 中的百分比。
- `fragmentation`：参见 [INFO](INFO.md) 的 `mem_fragmentation_ratio`。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：包含内存使用指标名称和对应值的嵌套列表。

**关于主页中使用的关键词 slave 的说明：** 为了保证 Redis 5 的向后兼容性，我们在 Redis 5 中继续使用关键词 slave。slave 是协议的一部分，我们会在 API 自然消亡后启用关键词 slave 的使用。