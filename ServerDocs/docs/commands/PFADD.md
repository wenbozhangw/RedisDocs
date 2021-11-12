## PFADD key [element [element ...]]

    起始版本：2.8.9.
    时间复杂度：每添加一个元素 O(1)。

将除了第一个参数以外的参数存储到以第一个参数为变量名的 HyperLogLog 结构中。

这个命令的一个副作用是它可能会更改这个 HyperLogLog 的内部来反应在每添加一个唯一的对象时估计的基数（集合的基数）。

如果一个 HyperLogLog 的估计的近似基数在执行命令过程中发生了变化，[PFADD](PFADD.md) 返回 1，否则返回 0，如果指定的 key 不存在，这个命令会自动创建一个空的 HyperLogLog 结构（指定长度和编码的字符串）。

如果在调用该命令时仅提供变量名，而不指定元素也是可以的，如果这个变量名存在，则不会有任何操作，如果不存在，则会创建一个数据结构（返回 1）。

了解更多 HyperLogLog 数据结构，请查阅 [PFCOUNT](PFCOUNT.md) 命令页面。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- 如果 HyperLogLog 的内部被修改了，那么返回 1，否则返回 0。

---

### Examples

```
redis> PFADD hll a b c d e f g
(integer) 1
redis> PFCOUNT hll
(integer) 7
redis> 
```