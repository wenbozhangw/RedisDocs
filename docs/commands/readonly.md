## READONLY

    自 3.0.0 起可用。
    时间复杂度：O(1)。

启用读取查询以连接到 Redis Cluster 副本节点(`replica node`)。

通常副本节点会将客户端重定向到给定命令中涉及的哈希槽的权威 master node，但是客户端可以使用副本来使用 [READONLY](readonly.md) 命令扩展读取。

[READONLY](readonly.md) 告诉 Redis Cluster 副本节点客户端愿意读取可能过时的数据并且对运行写入查询不感兴趣。

当连接处于只读模式时，仅当操作涉及副本的主节点未提供的 key 时，集群才会向客户的发送重定向。这可能是因为：
1. 客户端发送了一个关于这个副本的主节点从未提供过服务的哈希槽的命令。
2. 集群被重新配置（例如重新分片）并且副本不再能够为给定的哈希槽提供命令。

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)