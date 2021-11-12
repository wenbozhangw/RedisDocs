## PFCOUNT key [key ...]

    起始版本：2.8.9.
    时间复杂度：当使用单个 key 调用时，每个 key 具有很小的平均常数时间 O(1)。当使用多个 key 调用时，常数时间要大得多，为 O(N)，其中 N 是 key 的数量。

当参数为一个 key 时，返回存储在 HyperLogLog 结构体的该变量的近似基数，如果该变量不存在，则返回 0。

当参数为多个 key 时，返回这些 HyperLogLog 并集的近似基数，这个值是将所给定的所有 key 的 HyperLogLog 结构合并到一个临时的 HyperLogLog 结构中计算而得到的。

HyperLogLog 可以使用固定且很少的内存（每个 HyperLogLog 结构需要 12K 字节再加上 key 本身的几个字节）来存储集合的**唯一**元素。

返回的可见集合基数并不是精确值，而是一个带有 0.81% 标准错误(standard error) 的近似值。

例如为了记录一天会执行多少次各不相同的搜索查询，一个程序可以在每次执行搜索查询时调用一次 [PFADD](PFADD.md) ，并通过调用 [PFCOUNT](PFCOUNT.md) 命令来获取这个记录的近似结果。

注意：这个命令的副作用是可能会导致 HyperLogLog 内部结构被更改，处于缓存的目的，它会用 8 个字节来记录最近一次计算得到的基数，所以 [PFCOUNT](PFCOUNT.md) 命令在技术上是一个写命令。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- [PFADD](PFADD.md) 添加的唯一元素的近似数量。

---

### Examples

```
redis> PFADD hll foo bar zap
(integer) 1
redis> PFADD hll zap zap zap
(integer) 0
redis> PFADD hll foo bar
(integer) 0
redis> PFCOUNT hll
(integer) 3
redis> PFADD some-other-hll 1 2 3
(integer) 1
redis> PFCOUNT hll some-other-hll
(integer) 6
redis> 
```

---

### Performances

当调用 [PFCOUNT](PFCOUNT.md) 命令时，指定单个 key 作为参数，性能表现很好，甚至和处理一个 HyperLogLog 所需要的时间一样短。这可能和 [PFCOUNT](PFCOUNT.md) 命令能够直接使用缓存的估计基数有关，大多数的 [PFADD](PFADD.md) 也不会更新任何寄存器，所以这个值也很少被更改。理论上可以达到每秒上百次操作。

当调用 [PFCOUNT](PFCOUNT.md) 命令时，指定多个 key ，由于在多个 HyperLogLog 结构中执行一笔较慢合并操作，并且这个通过并集计算得到的基数是不能被缓存的，[PFCOUNT](PFCOUNT.md) 命令还要消耗毫秒量级的时间来进行多个 key 的并集操作，消耗的时间会比较长一些，所以不要滥用这种多个 key 的方式。

使用者需要明白这些命令处理一个 key 和多个 key 执行的语义是不同的，并且执行的性能也不相同。

---

### HyperLogLog representation

Redis HyperLogLogs 使用 double representation 来表示：稀疏表示(sparse representation)适用于计数少量元素（导致少量寄存器设置为非零值）的稀疏表示，以及适用于更高基数的密集表示(dense representation)。Redis 会在需要时自动从稀疏表示切换到密集表示。

稀疏表示使用经过优化的运行长度编码(run-length encoding)，以及有效的存储大量设置为零的寄存器。墨迹表示是一个 12288 bytes 的 Redis 字符串，已存储 16384 个 6 bit 计数器。对双重表示的需要来自这样一个事实，即使用 12K(这是密集表示内存要求)金编码几个寄存器以获得较小的基数是非常不好的。

两种表示都以一个 16 字节的标头为前缀，其中包括一个 magic，一个 encoding/version 字段和计算出的缓存基数估计，以小端格式存储（如果自 HyperLogLog 更新以来估计无效，则最高有效位为 1，因为计算了基数）。

HyperLogLog 是一个 Redis 字符串，可以使用 [GET](GET.md) 检索并使用 [SET](SET.md) 恢复。使用损坏的 HyperLogLog 调用 [PFADD](PFADD.md) 、[PFCOUNT](PFCOUNT.md) 或 [PFMERGE](PFMERGE.md) 命令从来都不是问题，他可能会返回随机值，但不会影响服务器的稳定性。大多数情况下，在损坏稀疏表示时，服务器会识别损坏并返回错误。

从处理器字长和字节序的角度来看，该表示是中性的，因此 32 位和 64 位处理器，大端或小端使用相同的表示。

有关 Redis HyperLogLog 实现的更多详细信息，请参阅 [this blog post](http://antirez.com/news/75) 。`hyperloglog.c` 文件中实现的源代码也很容易阅读和理解，并且包括用于稀疏和密集表示的精确编码的完整规范。