## FLUSHDB [ASYNC|SYNC]

    起始版本：1.0.0。
    时间复杂度：O(N)，其中 N 为选择的数据库中 key 的数量。

删除当前选定数据库的所有 key。这个命令永远不会失败。

默认情况下，[FLUSHDB](flushdb.md) 将同步刷新数据库中的所有 key。从 Redis 6.2 开始，将 `lazyfree-lazy-user-flush` 配置指令设置为 `yes` 将会把默认刷新模式改为异步。

可以使用以下修饰符之一来明确指定刷新模式：
- `ASYNC` : 异步刷新数据库
- `SYNC` : 同步刷新数据库

注意：异步 [FLUSHDB](flushdb.md) 命令仅删除调用该命令时存在的 key。在异步刷新期间创建的 key 将不受影响。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)