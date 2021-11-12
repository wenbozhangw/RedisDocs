## MSETNX key value [key value ...]

    起始版本：1.0.1.
    时间复杂度：O(N) 其中 N 为要设置的 key 个数。

将给定的 key 设置为其各自的值。即使只有一个 key 已经存在，[MSETNX](MSETNX.md) 也不会执行任何操作。

由于这种特性，[MSETNX](MSETNX.md) 可以实现要么所有的操作都成功，要么一个都不执行，这样可以用来设置不同的key，来表示一个唯一的对象的不同字段。

[MSETNX](MSETNX.md) 是原子的，所以所有给定的 key 都是一次设置的。客户端不可能看到某些 key 已经更新而其他 key 未更改。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- 1，如果所有的 key 都被设置
- 0，如果没有 key 被设置（至少其中有一个 key 是存在的）

---

### Examples

```
redis> MSETNX key1 "Hello" key2 "there"
(integer) 1
redis> MSETNX key2 "new" key3 "world"
(integer) 0
redis> MGET key1 key2 key3
1) "Hello"
2) "there"
3) (nil)
redis> 
```