## MIGRATE host port key|"" destination-db timeout [COPY] [REPLACE] [AUTH password] [AUTH2 username password] [KEYS key [key ...]]

    起始版本：2.6.0.
    时间复杂度：该命令实际上在源实例中执行 DUMP+DEL，在目标实例中执行 RESTORE。有关时间复杂度，请参阅这些命令的页面。还执行了两个实例之间的 O(N) 数据传输。

以原子方式将 key 从源 Redis 实例传输到目标 Redis 实例。成功后，key 将从源实例中删除，并保证存在于目标实例中。

该命令是原子的，并在传输 key 所需要的时间内阻塞两个实例，在任何给定时间，除非发生超时错误，否则 key 将出现在给定实例或另一个示例中。在 3.2 及更高版本中，通过传递空字符串("")左移 key 并添加 [KEYS](keys.md) 子句，可以在堆 [MIGRATE](migrate.md) 的单个调用中流水线化处理多个 key。

该命令在内部使用 [DUMP](dump.md) 生成键值的序列化版本，并使用 [RESTORE](restore.md) 来合成目标实例中的 key。源实例充当目标实例的客户端。如果目标实例对 [RESTORE](restore.md) 命令返回 OK，则源实例使用 [DEL](del.md) 删除 key。

`timeout` 参数以毫秒为单位，指定当前实例和目标实例进行沟通的最大间隔时间。这说明操作不一定要在 `timeout` 毫秒内完成，但传输应该在不超过指定的毫秒数的情况下进行。

[MIGRATE](migrate.md) 需要在给定的超时时间内完成 I/O 操作。当传输过程中出现 I/O 错误或达到超时时间时，操作将终止并返回特殊错误 - _IOERR_ 。发生这种情况时，可能会出现以下两种情况：
- key 可能存在于两个实例中。
- key 可能只存在于源实例中。

key 不可能在超时的情况下丢失，但是调用 [MIGRATE](migrate.md) 的客户端在超时错误的情况下应该检查 key 是否也存在于目标实例中并采取相应的行动。

当返回任何其他错误（以 ERR 开头）时，[MIGRATE](migrate.md) 保证该 key 仍然仅存在于源实例中（除非具有相同名称的 key 也存在于弥补实例中）。

如果源实例中没有要迁移的 key，则返回 _NOKEY_。因为在正常情况下丢失 key 是可能的，例如到期， _NOKEY_ 不是错误。

---

### Migrating multiple keys with a single command call

从 Redis 3.0.6 开始，[MIGRATE](migrate.md) 支持新的批量迁移模式，该命令使用流水线以便在实例之间迁移多个 key，而不会产生往返时间延迟和其他开销，而这些开销是通过单个 [MIGRATE](migrate.md) 调用移动每个 key 时产生的。

为了启用这种形式，使用了 `KEYS` 选项，并将普通的 key 参数设置为空字符串。实际的 key 名将在 `KEYS` 参数本身之后提供，如下例所示：

```
MIGRATE 192.168.1.34 6379 "" 0 5000 KEYS key1 key2 key3
```

使用这种形式时，仅当实例中不存在任何 key 时，餐返回 _NOKEY_ 状态代码，否则执行命令，即使只存在一个 key。

---

### Options

- `COPY` - 不要从本地实例中删除 key。
- `REPLACE` - 替换远程实例上的现有 key。
- `KEYS` - 如果 key 参数是空字符串，则该命令将迁移所有跟在 [KEYS](keys.md) 选项之后的 key（有关更多信息，请参阅上一节）。
- `AUTH` - 使用给定的密码对远程实例进行身份验证。
- `AUTH2` - 使用给定的用户名和密码进行身份验证（Redis 6 或更高版本的 ACL 身份验证样式）。

---

### History

- &gt;= 3.0.0：增加 `COPY` 和 `REPLACE` 选项。
- &gt;= 3.0.6：增加 `KEYS` 选项。
- &gt;= 4.0.7：增加 `AUTH` 选项。
- &gt;= 6.0.0：增加 `AUTH2` 选项。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：该命令在成功时返回 OK，如果在源实例中没有找到 key，则返回 NOKEY。