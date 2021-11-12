## SELECT index

    起始版本：1.0.0。

选择具有指定的从零开始的数字索引的 Redis logical database。新连接始终使用 database 0。

可选的 Redis 数据库是一种命名空间形式：所有数据库仍然保存在同一个 RDB/AOF 文件中。但是，不同的数据库可以具有相同名称的 key，而且比如 [FLUSHDB](FLUSHDB.md)、[SWAPDB](SWAPDB.md) 或 [RANDOMKEY](RANDOMKEY.md) 之类的命令适用于特定数据库。

实际上，Redis 数据库应该用于分离属于同一应用程序的不同 key（如果需要），而不是使用单一的 Redis 实例为多个不同的应用程序提供不同数据库。

使用 Redis Cluster 时，不能使用 [SELECT](select.md) 命令，因为 Redis Cluster 只支持 database 0。在 Redis Cluster 情况下，拥有多个数据库将毫无用处，并且会产生不必要的复杂度。对于 Redis Cluster 设计和目标，在单个数据库上原子操作的命令是不可能的。

由于当前选择的数据库是 connection 的一个属性，客户端应该跟踪当前选择的数据库并在重新连接时重新选择它。虽然没有命令可以查询当前连接的数据库，但 [CLIENT LIST](CLIENT LIST.md) 输出显示每个客户端当前选定的数据库。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) 。