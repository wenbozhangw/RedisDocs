## MSET key value [key value ...]

    起始版本：1.0.1.
    时间复杂度：O(N) 其中 N 是要设置的 key 数量。

将给定的 key 设置为其各自的值。[MSET](mset.md) 用新值替换现有值，就像常规 [SET](set.md) 一样。如果您不想覆盖现有值，请参阅 [MSETNX](msetnx.md) 。

[MSET](mset.md) 是原子的，所以所有给定的 key 都是一次设置的。客户端不可能看到某些 key 已更新而其他 key 未更改。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)：总是返回 OK，因为 [MSET](mset.md) 不会失败。

---

### Examples

```
redis> MSET key1 "Hello" key2 "World"
"OK"
redis> GET key1
"Hello"
redis> GET key2
"World"
redis> 
```