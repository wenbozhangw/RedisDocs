## BITOP operation destkey key [key ...]

    起始版本：2.6.0.
    时间复杂度：O(N)

对一个或多个保存二进制位的字符串 key 进行位运算，并将结果存放在 destkey 中。

[BITOP](bitop.md) 命令支持 **AND、OR、XOR、NOT** 这四种操作中的任意一种参数：
- BITOP AND destkey srckey1 srckey2 srckey3 ... srckeyN
- BITOP OR destkey srckey1 srckey2 srckey3 ... srckeyN
- BITOP XOR destkey srckey1 srckey2 srckey3 ... srckeyN
- BITOP NOT destkey srckey

除了 **NOT** 操作之外，其他操作都可以接收一个或多个 key 作为输入。

执行结果将始终保存在 destkey 里面。

---

### Handling of strings with different lengths

当在具有不同长度的字符串之间执行操作时，所有比集合中最长字符串短的字符串都被视为零填充，直到最长字符串长度。

对于不存在的 key 也是如此，它们被认为是最长字符串长度的 0 字符串序列。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)

保存到 destkey 的字符串的长度，和输入 key 中最长的字符串的长度相等。

---

### Examples

```
redis> SET key1 "foobar"
"OK"
redis> SET key2 "abcdef"
"OK"
redis> BITOP AND dest key1 key2
(integer) 6
redis> GET dest
"`bc`ab"
redis> 
```

---

### Pattern: real time metrics using bitmaps

[BITOP](bitop.md) 是对 [BITCOUNT](bitcount.md) 命令一个很好的补充。不同的 bitmaps 进行组合操作可以获得目标 bitmap 以进行人口统计操作。

[Fast easy realtime metrics using Redis bitmaps](http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps) 这篇文章介绍了一个有趣的用例。

---

### Performance considerations

[BITOP](bitop.md) 可能是一个缓慢的命令，它的时间复杂度是O(N)。 在处理长字符串时应注意一下效率问题。

对于实时的指标和统计，涉及大输入一个很好的方法是 使用bit-wise操作以避免阻塞主实例。