## PEXPIRE key milliseconds [NX|XX|GT|LT]

    起始版本：2.6.0。
    时间复杂度：O(1)。

这个命令和 [EXPIRE](EXPIRE.md) 命令的作用类似，但是它以毫秒为单位设置 key 的生存时间，而不像 [EXPIRE](EXPIRE.md) 命令哪有，以秒为单位。

---

### Options

[PEXPIRE](PEXPIRE.md) 命令从 Redis 7.0 开始支持一组选项：

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
redis> PEXPIRE mykey 1500
(integer) 1
redis> TTL mykey
(integer) 1
redis> PTTL mykey
(integer) 1497
redis> PEXPIRE mykey 1000 XX
ERR ERR wrong number of arguments for 'pexpire' comman
```

```
redis> TTL mykey
(integer) 1
redis> PEXPIRE mykey 1000 NX
ERR ERR wrong number of arguments for 'pexpire' command
```

```
redis> TTL mykey
(integer) 1
redis> 
```

---

### History

- &gt;= 7.0：增加选项：NX, XX, GT 和 LT。