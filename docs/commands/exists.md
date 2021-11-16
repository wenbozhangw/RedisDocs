## EXISTS key [key ...]

    起始版本：1.0.0。
    时间复杂度：O(N) 其中 N 是要检查的 key 数。

返回 key 是否存在。

从 Redis 3.0.3 开始，可以指定多个 key 而不是单个 key。在这种情况下，他返回存在的 key 的数量。请注意，单个 key 返回 1 或 0 只是可变参数用法的一种特殊情况，因此该命令完全向后兼容。

用户应该知道，如果在参数中多次提到相同的 key，他将被多次计算。所以如果 somekey 存在，`EXISTS somekey somekey` 将返回 2。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) ，特别的 :
- 1，如果指定的 key 存在。
- 0，如果指定的 key 不存在。

自 Redis 3.0.3 开始，该命令接受可变数量的 key，并且返回值是通用的：
- 指定为参数的 key 中存在的 key 的数量。多次提及和存在的 key 会被多次计数。

---

### Examples

```
redis> SET key1 "Hello"
"OK"
redis> EXISTS key1
(integer) 1
redis> EXISTS nosuchkey
(integer) 0
redis> SET key2 "World"
"OK"
redis> EXISTS key1 key2 nosuchkey
(integer) 2
redis> 
```