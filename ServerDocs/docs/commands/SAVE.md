## SAVE

    起始版本：1.0.0.

[SAVE](SAVE.md) 命令执行一个 **同步(synchronous)** 保存操作，以 RDB 文件的形式生成 Redis 实例内所有数据的时间点快照。

您几乎不想在生产环境中调用 [SAVE](SAVE.md)，因为它会阻塞所有其他客户端。相反，通常会使用 [BGSAVE](BGSAVE.md)。但是，若果在 [BGSAVE](BGSAVE.md) 命令的保存数据的子进程发生错误时（例如 fork(2) 系统调用中的错误），用 [SAVE](SAVE.md) 命令可能是执行最新数据集转储的最后手段。

有关详细信息，请参阅 [persistence documentation](../topics/persistence.md)。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：命令成功返回 OK。
