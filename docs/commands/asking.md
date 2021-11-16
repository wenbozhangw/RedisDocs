## ASKING

    起始版本：3.0.0.
    时间复杂度：O(1)

当集群客户端收到 `-ASK` 重定向时，[ASKING](asking.md) 命令将发送到目标节点，然后是重定向的命令。这通常由集群客户端自动完成。

如果在事务过程中收到 `-ASK` 重定向，则只需向目标节点发送一个 ASKING 命令，即可将完整的交易发送到目标节点。

有关详细信息，请参阅 [ASK redirection in the Redis Cluster Specification](../topics/cluster-spec.md#ask-redirection) 。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：OK.