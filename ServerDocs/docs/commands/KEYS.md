## KEYS pattern

    起始版本：1.0.0.
    时间复杂度：O(N), 其中 N 是数据库中的 key 数量，假设数据库中的 key 名和给定的 pattern 长度有限。

返回所有匹配给定 `pattern` 的key。

虽然此操作的时间复杂度为 O(N)，但常数时间相当低。例如，运行在入门级笔记本电脑上的 Redis 可以在 40 毫秒内扫描 100 万个数据库 key 。

**警告：** 将 [KEYS](KEYS.md) 视为仅应在生成环境中及其小心地使用的命令。当它针对大型数据库执行时，他可能会破坏性能。此命令用户调试和特殊操作，例如改变键空间的布局。不要在常规应用程序代码中使用 [KEYS](KEYS.md)。如果您正在寻找一种在键空间子集中查找键的方法，请考虑使用 [SCAN](SCAN.md) 或 [sets](../topics/data-types.md#sets) 。

支持的全局样式模式：
- `h?llo 匹配 hello, hallo 和 hxllo`
- `h*llo 匹配 hllo 和 heeeello`
- `h[ae]llo 匹配 hello 和 hallo, 但是不匹配 hillo`
- `h[^e]llo 匹配 hallo, hbllo, … 但是不匹配 hello`
- `h[a-b]llo 匹配 hallo 和 hbllo`

如果你想取消字符的特殊匹配（正则表达式），可以在它的前面加 `\`。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：所有匹配 `pattern` 的 key 列表。

---

### Examples

```
redis> MSET firstname Jack lastname Stuntman age 35
"OK"
redis> KEYS *name*
1) "lastname"
2) "firstname"
redis> KEYS a??
1) "age"
redis> KEYS *
1) "lastname"
2) "firstname"
3) "age"
redis> 
```
