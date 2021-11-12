## BITPOS key bit [start [end]]

    起始版本：2.8.7.
    时间复杂度：O(N)。

返回字符串里面第一个被设置为 1 或者 0 的 bit位置。

返回一个位置，把字符串当做一个从左到右的字节数组，其中第一个字节的最高有效位位于位置 0，第二个字节的最高有效位位于位置 8，以此类推。

[GETBIT](GETBIT.md) 和 [SETBIT](SETBIT.md) 相似的也是操作字节 bit 的命令。

默认情况下，整个字符串都会被检索一次。只有在指定 start 和 end 参数（指定 _start_ 和 _end_ 位是可行的），该范围被解释为一个字节的范围，而不是一系列的位。所以 start = 0 并且 end = 2 是指前三个字节范围内查找。

注意，返回的位的位置始终是从 0 开始的，即使使用了 start 来指定的一个开始字节也是这样。

和 [GETRANGE](GETRANGE.md) 命令一样，start 和 end 也可以包含负值，负值将从字符串的末尾开始计算，-1 是字符串的最后一个字节，-2 是倒数第二个，等等。

不存在的 key 将会被当做空字符串来处理。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)

命令返回字符串里面第一个被设置为 1 或者 0 的 bit 位置。

如果我们在空字符串或者 0 字节的字符串里面查找 bit 为 1 的内容，那么结果将返回 -1 。

如果我们在字符串里面查找 bit 为 0 而且字符串只包含 1 的值时，将返回字符串最右边的第一个空位。如果有一个字符串是三个字节的值为 0xff 的字符串，那么命令 `BITPOS key 0` 将会返回 24，因为 0-23 位都是 1。

基本上，我们可以把字符串看成右边有无数个 0 。

然而，如果你用指定 start 和 end 范围进行查找指定值时，如果该范围内没有对应值，结果将返回 -1。

---

### Examples

```
redis> SET mykey "\xff\xf0\x00"
"OK"
redis> BITPOS mykey 0
(integer) 12
redis> SET mykey "\x00\xff\xf0"
"OK"
redis> BITPOS mykey 1 0
(integer) 8
redis> BITPOS mykey 1 2
(integer) 16
redis> set mykey "\x00\x00\x00"
"OK"
redis> BITPOS mykey 1
(integer) -1
redis> 
```