## SCRIPT FLUSH [ASYNC|SYNC]

    起始版本：2.6.0。
    时间复杂度：O(N)，其中 N 为缓存中脚本的数量。

刷新 Lua 脚本缓存。

有关 Redis Lua 脚本的详细信息，请参阅 [EVAL](eval.md) 文档。

默认情况下，[SCRIPT FLUSH](script-flush.md) 将同步刷新缓存。从 Redis 6.2 开始，将 **lazyfree-lazy-user-flush** 配置指令设置为 "yes" 会将默认刷新模式更改为异步。

可以使用以下修饰符之一来明确指定刷新模式：
- `ASYNC` : 异步刷新缓存
- `SYNC` : 异步刷新缓存

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)

---

### History

- &gt;= 6.2.0: 添加了 `ASYNC` 和 `SYNC` 刷新模式修饰符，以及 `lazyfree-lazy-user-flush` 配置指令。