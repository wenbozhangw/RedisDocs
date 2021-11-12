## READWRITE

    自 3.0.0 起可用。
    时间复杂度：O(1)。

禁用连接到 Redis Cluster 从节点 (`slave node`) 的读取查询。

默认情况下禁用对 Redis Cluster 从节点的读取查询，但您可以使用 [READONLY](READONLY.md) 命令在每个连接的基础上更改此行为。[READWRITE](READWRITE.md) 命令将连接的 readonly mode flag 重置回 readwrite。

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)
