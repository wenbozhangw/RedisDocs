## SCRIPT KILL

    起始版本：2.6.0。
    时间复杂度：O(1)。

杀死当前正在运行的 Lua 脚本，当且仅当这个脚本没有执行过任何写操作时，这个命令才生效。

此命令主要用于终止运行时间过长的脚本，比如一个因为 BUG 而发生无限循环的脚本。该脚本将被终止，当前被执行 EVAL 阻塞的客户端将看到命令返回并显示错误。

如果脚本已经执行了写操作，则不能以这种方式杀死他，因为这会违反 Lua 脚本原子性执行原则。在这种情况下，唯一可行的办法是使用 `SHUTDOWN NOSAVE` 命令，通过停止整个 Redis 进程来停止脚本的运行，病防治不完整(half-written)的信息被写入数据库。

有关 Redis Lua 脚本的详细信息，请参阅 [EVAL](eval.md) 文档。

---


### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)