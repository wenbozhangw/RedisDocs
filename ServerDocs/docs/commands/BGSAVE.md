## BGSAVE [SCHEDULE]

    起始版本：1.0.0.

后台保存 DB。

通常立即返回 OK 状态码。Redis forks，父进程继续为客户端服务，子进程将数据库保存在磁盘上然后退出。

如果已经有后台保存在运行，或者如果有另一个非后台保存进程在运行，特别是正在进行的 AOF 重写，则会返回错误。

如果使用 `BGSAVE SCHEDULE`，该命令将在 AOF 重写正在进行时立即返回 OK 并安排后台保存在下一次机会运行。

客户端可以使用 [LASTSAVE](LASTSAVE.md) 命令价差操作是否成功。

有关详细信：，请参阅 [persistence documentation](../topics/persistence.md)。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：如果 [BGSAVE](BGSAVE.md) 正确启动 `Background saving started`，或当使用 `SCHEDULE` 子命令时，`Background saving scheduled`。

---

### History

- &gt;= 3.2.2：增加 `SCHEDULED` 选项。