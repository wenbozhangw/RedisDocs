## PUBSUB CHANNELS [pattern]

    起始版本：2.8.0.
    时间复杂度：O(N) 其中 N 是活动的频道的数量，并假设恒定时间模式匹配（相对较短的通道和模式）。

返回当前 _active channels_ 的列表。

活动频道是具有一个或多个订阅者（不包括订阅模式的客户端）的 Pub/Sub 频道。

如果未指定 `pattern`，则列出所有频道，否则如果指定了 `pattern`，则仅列出与指定的 glob-style pattern 匹配的频道。

集群注意：在 Redis Cluster 中，客户端可以订阅每个节点，也可以发布到其他每个节点。集群将确保根据需要转发已发布的消息。也就是说，集群中 PUBSUB 的回复仅报告来自节点的 Pub/Sub 上下文的信息，而不是整个集群。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：活动频道列表，可选择匹配指定的模式。