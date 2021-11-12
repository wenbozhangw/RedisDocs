## EXPIREAT key timestamp [NX|XX|GT|LT]

    起始版本：1.2.0。
    时间复杂度：O(1)。

[EXPIREAT](EXPIREAT.md) 的作用和 [EXPIRE](EXPIRE.md) 类似，但不同的是， 它不是指定代表 TTL(time to live) 的秒数，而是采用绝对 [Unix timestamp](http://en.wikipedia.org/wiki/Unix_time) （自 1970 年 1 月 1 日以来的秒数）。过去的时间戳将立即删除 key。

命令的具体语义请参考 [EXPIRE](EXPIRE.md) 的文档。

---

### Background

引入 [EXPIREAT](EXPIREAT.md) 是为了将相对过期时间转换为 AOF 持久化模式的绝对超时时间。 当然，也可以直接用它来指定给定 key 应在未来的指定事件过期。

---

### Options

[EXPIREAT](EXPIREAT.md) 命令从 Redis 7.0 开始支持一组选项：

- NX —— 仅当 key 没有过期时间时，才设置过期时间。
- XX —— 仅当 key 存在过期时间时，才设置过期时间。
- GT —— 仅当新的过期时间大于当前过期时间时，才设置过期时间。
- LT —— 仅当新的过期时间小于当前过期时间时，才设置过期时间。

处于 GT 和 LT 的目的，非易变的(non-volatile) key 被视为无限的 TTL。GT、LT 和 NX 选项是互斥的。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- 1，设置成功。
- 0，如果设置超时失败。例如键不存在，或由于提供的参数而跳过操作。
---

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> EXISTS mykey
(integer) 1
redis> EXPIREAT mykey 1293840000
(integer) 1
redis> EXISTS mykey
(integer) 0
redis> 
```

---

### History

- &gt;= 7.0：增加选项：NX, XX, GT 和 LT。