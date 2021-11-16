## PFMERGE destkey sourcekey [sourcekey ...]

    起始版本：2.8.9.
    时间复杂度：O(N)，其中 N 为合并 HyperLogLog 的个数，但具有很高的常数时间。

将多个 HyperLogLog 值合并为一个唯一值，该值将近似于观察到的源 HyperLogLog 结构集合的并集的基数。

计算出的合并后的 HyperLogLog 设置为目标变量，如果不存在则创建改变量（默认为空的 HyperLogLog）。

如果目标变量存在，则将其视为源集合之一，其中基数将包含在计算的 HyperLogLog 的基数中。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：此命令只会返回 OK。

---

### Examples

```
redis> PFADD hll1 foo bar zap a
(integer) 1
redis> PFADD hll2 a b c foo
(integer) 1
redis> PFMERGE hll3 hll1 hll2
"OK"
redis> PFCOUNT hll3
(integer) 6
redis> 
```