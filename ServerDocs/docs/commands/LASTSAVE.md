## LASTSAVE

    起始版本：1.0.0.

返回上次成功执行的 DB 保存的 `UNIX TIME`。客户端可以检查 [BGSAVE](BGSAVE.md) 命令是否成功读取 [LASTSAVE](LASTSAVE.md) 值，然后发出 [BGSAVE](BGSAVE.md) 命令并每隔 N 秒定期检查 [LASTSAVE](LASTSAVE.md) 是否更改。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：一个 UNIX 时间戳。