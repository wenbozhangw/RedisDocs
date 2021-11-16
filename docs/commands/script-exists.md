## SCRIPT EXISTS sha1 [sha1 ...]

    起始版本：2.6.0
    时间复杂度：O(N)，其中 N 是要检查的脚本数量（单个检查脚本操作是 O(1)）。

返回有关脚本缓存(script cache)中脚本存在的信息。

此命令接收一个或多个 SHA1 摘要并返回一列 1 或 0 以表示脚本是否已经在脚本缓存中定义。这在流水线操作之前很有用，以确保加载脚本（如果没有，则使用[SCRIPT LOAD](script-load.md) 加载它们），以便可以单独使用 [EVALSHA](evalsha.md) 而不是 [EVAL](eval.md) 来执行流水线操作以节省带宽。

有关 Redis Lua 脚本的详细信息，请参阅 [EVAL](eval.md) 文档。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) ：该命令返回与指定的 SHA1 摘要参数对应的整数数组。对于脚本对应的 SHA1 摘要在脚本缓存中实际存在，返回 1，否则返回 0。