## SETRANGE key offset value

    起始版本：2.2.0.
    时间复杂度：O(1)，不计算将新字符串复制到指定位置所需的时间。通常，该字符串非常小，因此平摊时间复杂度为 O(1)。否则，复杂度为 O(M)，其中 M 是 value 参数的长度。

这个命令的作用是覆盖 _key_ 对应的 string 的一部分，从指定的 offset 处开始，覆盖 _value_ 的长度。如果 offset 比当前 key 对应 string 还要长，那这个 string 后面就补 0( padded with zero-bytes)以达到 offset。

不存在的 keys 被认为是空字符串，所以这个命令可以确保 key 有一个足够大的字符串，能在 offset 处设置 value。

注意，offset 最大可以是 `2<sup>29</sup> - 1 (536870911)`，因为 Redis 字符串限制在 512M 大小。如果你需要超过这个大小，你可以用多个 keys。

**警告**：当 set 最后一个字节并且 key 还没有一个字符串值，或者其值是一个比较小的字符串时，Redis 需要立即分配所有内存，这有可能会导致服务阻塞一会儿。在一台 2010 MacBook Pro 上，设置 536870911 bytes (分配 512MB)需要 ~300ms，设置 134217728 bytes (分配128MB)需要 ～80ms，设置 33554432 bits (分配 32MB) 需要 30ms，设置 8388608 bits (分配 8MB) 需要 8ms。注意，一旦第一次内存分配完，后面对同一个 key 调用 [SETRANGE](setrange.md) 就会预先得到内存分配。

---

### Patterns

正因为有了 [SETRANGE](setrange.md) 和类似功能的 [GETRANGE](getrange.md) 命令，你可以把 Redis 的字符串当成线性数组，随机访问只要 O(1) 的复杂度。这在很多真实场景应用里非常快而且高效。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：该命令修改后的字符串长度。

---

### Examples

基本使用方法：

```
redis> SET key1 "Hello World"
"OK"
redis> SETRANGE key1 6 "Redis"
(integer) 11
redis> GET key1
"Hello Redis"
redis> 
```

补 0 的例子：

```
redis> SETRANGE key2 6 "Redis"
(integer) 11
redis> GET key2
"\u0000\u0000\u0000\u0000\u0000\u0000Redis"
redis> 
```