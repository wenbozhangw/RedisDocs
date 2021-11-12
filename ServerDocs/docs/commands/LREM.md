## LREM key count element

    起始版本：1.0.0
    时间复杂度：O(N + M) 其中 N 是列表的长度，M 是删除的元素数。

从存储在 key 对应的列表中删除前 count 次出现的值与 `element` 相等的元素。`count` 参数通过下面几种方式影响这个操作：

- `count > 0`：从头到尾移除值为 `element` 的元素。
- `count < 0`：从尾到头移除值为 `element` 的元素。
- `count = 0`：移除所有值为 `element` 的元素。

比如，`LREM list -2 "hello"` 会从存在于 list 的列表中移除最后两个出现的 "hello"。

需要注意的是，如果 list 对应的 key 不存在，会被当做空列表处理，所以当 key 不存在的时候，这个命令会返回 0。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：被移除的元素个数。

---

### Examples

```
redis> RPUSH mylist "hello"
(integer) 1
redis> RPUSH mylist "hello"
(integer) 2
redis> RPUSH mylist "foo"
(integer) 3
redis> RPUSH mylist "hello"
(integer) 4
redis> LREM mylist -2 "hello"
(integer) 2
redis> LRANGE mylist 0 -1
1) "hello"
2) "foo"
redis> 
```