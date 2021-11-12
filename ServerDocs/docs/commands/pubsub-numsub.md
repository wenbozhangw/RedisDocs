## PUBSUB NUMSUB [channel [channel ...]]

    起始版本：2.8.0.
    时间复杂度：NUMSUB 子命令为 O(N)，其中 N 是请求的通道数量。

返回指定频道(channel)的订阅者数量（不包括订阅模式的客户端）。

需要注意的是，在没有频道的情况下调用命令是有效的。在这种情况下，他只会返回一个空列表。

集群注意：在 Redis Cluster 中，客户端可以订阅每个节点，也可以发布到其他每个节点。集群将确保根据需要转发已发布的消息。也就是说，集群中 PUBSUB 的回复仅报告来自节点的 Pub/Sub 上下文的信息，而不是整个集群。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：每个频道的频道和订阅者数量列表。

格式是 channel, count, channel, count, ..., 所以列表是扁平的。列出通道的顺序与命令调用中指定的通道顺序相同。