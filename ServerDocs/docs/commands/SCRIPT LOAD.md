## SCRIPT LOAD script

    起始版本：2.6.0。
    时间复杂度：O(N)，其中 N 为是脚本主体的字节长度。

将脚本添加到脚本缓存中，但不立即执行脚本。在脚本被加入到缓存紫华，通过 [EVALSHA](EVALSHA.md) 命令，可以使用脚本的 SHA1 摘要来调用这个脚本，就像第一次成功调用 [EVAL](EVAL.md) 之后一样。

脚本可以在缓存中保留无限长时间（直到执行 **SCRIPT FLUSH** 为止）。

即使脚本已经存在于脚本缓存中，该命令也会以相同的方式工作。

有关 Redis Lua 脚本的详细信息，请参阅 [EVAL](EVAL.md) 文档。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 此命令返回添加到脚本缓存中的脚本的 SHA1 摘要。