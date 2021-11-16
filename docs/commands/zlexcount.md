## ZLEXCOUNT key min max

    起始版本：2.8.9.
    时间复杂度：O(log(N)) 其中 N 是 sorted set 的元素数量。

当 sorted set 中的所有元素都以相同分数插入时，为了强制按照 lexicographical 排列，此命令返回  key 对应的 sorted set 中的元素数量，其值介于 min 和 max 之间。

min 和 max 参数的含义与 [ZRANGEBYLEX](zrangebylex.md) 中描述的含义相同。

注意：该命令的复杂度仅为 O(log(N))，因为它使用元素等级（参见 [ZRANK](zrank.md) ）来了解范围。因此，不需要做与范围大小成比例的工作。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：指定分数范围内的元素数量。

---

### Examples

```
redis> ZADD myzset 0 a 0 b 0 c 0 d 0 e
(integer) 5
redis> ZADD myzset 0 f 0 g
(integer) 2
redis> ZLEXCOUNT myzset - +
(integer) 7
redis> ZLEXCOUNT myzset [b [f
(integer) 5
redis> 
```