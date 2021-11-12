## PERSIST key

    起始版本：2.2.0。
    时间复杂度：O(1)。

移除给定 key 的生存时间，将这个 key 从 **易变的(volatile)**（带过期时间的key）转换成 **持久的(persistent)**（一个不带过期时间，永不过期的key）。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- 1，如果过期时间被移除。
- 0，key 不存在或没有关联的过期时间。
---

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> EXPIRE mykey 10
(integer) 1
redis> TTL mykey
(integer) 10
redis> PERSIST mykey
(integer) 1
redis> TTL mykey
(integer) -1
redis> 
```
