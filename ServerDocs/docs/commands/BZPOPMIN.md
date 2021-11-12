## BZPOPMIN key [key ...] timeout

    起始版本：5.0.0.
    时间复杂度：O(log(N))，其中 N 是 sorted set 中的元素数。

[BZPOPMIN](BZPOPMIN.md) 是 sorted set 命令 [ZPOPMIN](ZPOPMIN.md) 带有阻塞功能的版本。

在参数中的所有 sorted set 均为空的情况下，阻塞连接。参数中包含多个 sorted set 时，按照参数中 key 的顺序，返回第一个非空集合中分数最小的成员和对应的分数。

参数 `timeout` 可以理解为客户端被阻塞的最大毫秒数值，0 表示永久阻塞。

详细说明请参照 [BLPOP documentation](BLPOP.md)，[BZPOPMIN](BZPOPMIN.md) 使用 sorted set 类型的 key，[BLPOP](BLPOP.md) 使用 list 类型的 key，除此之外，两条命令无其他差别。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：

- 当 sorted set 无结果返回且超过 timeout 时，返回 nil。
- 返回三个元素的 multi-bulk 结果，第一元素为 key 名称，第二元素为 member 名称，第三元素为 score。

---

### History

- &gt;= 6.0：`timeout` 使用 double 代替 integer。

---

### Examples

```
redis> DEL zset1 zset2
(integer) 0
redis> ZADD zset1 0 a 1 b 2 c
(integer) 3
redis> BZPOPMIN zset1 zset2 0
1) "zset1"
2) "a"
3) "0"
```