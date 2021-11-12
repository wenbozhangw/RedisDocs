## BZPOPMAX key [key ...] timeout

    起始版本：5.0.0. 
    时间复杂度：O(log(N))，其中 N 为 sorted set 中的元素数量。

[BZPOPMAX](BZPOPMAX.md) 是 sorted set 命令 [ZPOPMAX](ZPOPMAX.md) 带有阻塞功能的版本。

在它是阻塞版本，因为当没有成员可以从任何给定的 sorted set 中弹出时，它会阻塞连接。从第一个非空的 sorted set 中弹出得分最高的成员，并按照给定的顺序检查给定的 key。

`timeout` 参数被解释为一个 double 值，指定要阻塞的最大秒数。0 表示永久阻塞。

详情说明请参考 [BZPOPMIN documentation](BZPOPMIN.md)，[BZPOPMAX](BZPOPMAX.md) 返回非空 sorted set key 中分数最大的成员，而 [BZPOPMIN](BZPOPMIN.md) 返回该 key 中分数最小的成员，除此之外，两条命令无其他差别。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：
- 当 sorted set 无结果且超时时间到达时，返回 nil multi-bulk reply。
- 返回三元素 multi-bulk 结果，第一个元素 key 名称，第二个元素成员名称，第三个元素分数。

---

### History

- &gt;= 6.0：`timeout`参数从 integer 替换为 double。

---

### Examples

```
redis> DEL zset1 zset2
(integer) 0
redis> ZADD zset1 0 a 1 b 2 c
(integer) 3
redis> BZPOPMAX zset1 zset2 0
1) "zset1"
2) "c"
3) "2"
```