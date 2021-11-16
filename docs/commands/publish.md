## PUBLISH channel message

    起始版本：2.0.0。
    时间复杂度：O(N + M)，其中 N 是订阅接受频道的客户端数量，M 是订阅模式的总数（任何客户端）。

向指定频道发布消息。

在 Redis 集群中，客户端可以发布消息到每个节点。集群确保根据需要，转发已发布的消息，因此客户端可以通过连接到任何一个节点来订阅任何 channel。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 收到消息的客户端数量。请注意，在 Redis Cluster 中，仅将发布客户端连接到同一节点的客户端计入计数。