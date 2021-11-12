## MEMORY PURGE

    起始版本：4.0.0.

命令 [MEMORY PURGE](MEMORY%20PURGE.md) 尝试清除脏页以便内存分配器回收使用。

该命令目前仅实现了 **jemalloc** 作为内存分配器的内存统计，并且对所有其他命令评估为良性 NOOP。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)