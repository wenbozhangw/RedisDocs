## PUBSUB NUMPAT

    起始版本：2.8.0.
    时间复杂度：O(1)。

返回客户端订阅的唯一模式 (unique pattern) 数量（使用 [PSUBSCRIBE](psubscribe.md) 命令执行）。

请注意，这不是订阅模式的客户端数量，而是所有客户端订阅的唯一模式总数量。

集群注意：在 Redis Cluster 中，客户端可以订阅每个节点，也可以发布到其他每个节点。集群将确保根据需要转发已发布的消息。也就是说，集群中 PUBSUB 的回复仅报告来自节点的 Pub/Sub 上下文的信息，而不是整个集群。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：所有客户端订阅的模式数量。