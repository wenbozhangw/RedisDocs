## ZINCRBY key increment member

    起始版本：1.2.0.
    时间复杂度：O(log(N)) 其中 N 是 Sorted sets 中的元素数量。

把 `key` 对应的 sorted sets 的成员(`member`)的分数(score)值增加 `increment`。如果 key 对应的 sorted set 不存在 member，则将其添加一个 member，score 为 increment（就好像它之前的 score 为 0.0）。若果 key 不存在，就创建一个只含有指定 member 的 sorted set。

当 key 不是 sorted set 类型时，返回一个错误。

score 值必须是字符串表示的整数值或双精度浮点数，并且能够接受 double 精度的浮点数。也有可能给以负数来减少 score 的值。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)：member 的新 score（双精度浮点数），用字符串表示。

---

### Examples

```
redis> ZADD myzset 1 "one"
(integer) 1
redis> ZADD myzset 2 "two"
(integer) 1
redis> ZINCRBY myzset 2 "one"
"3"
redis> ZRANGE myzset 0 -1 WITHSCORES
1) "two"
2) "2"
3) "one"
4) "3"
redis> 
```