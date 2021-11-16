## BRPOP key [key ...] timeout

    自 2.0.0 起可用。
    时间复杂度：O(N)，其中 N 为提供的 key 的个数。

[BRPOP](brpop.md) 是一个阻塞的列表弹出原语。它是 [RPOP](rpop.md) 的阻塞版本，因为这个命令会在给定 list 无法弹出任何元素的时候阻塞连接。该命令会按照给出的 key 顺序查看 list，并在找到的第一个非空 list 的尾部弹出一个元素。

有关更为准确的语义，请参阅 [BLPOP documentation](blpop.md) ，因为 [BRPOP](brpop.md) 与 [BLPOP](blpop.md) 相同，唯一的区别是它从 list 的尾部弹出元素而不是从 list 的头部弹出。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) ： 具体来说：
- 当没有元素可以弹出并且超过过期时间时，返回一个 nil multi-bulk。
- 当有元素弹出时会返回一个 two-element 的 multi-bulk，其中第一个元素是弹出元素的 key，第二个元素是 value。

---

### History

- &gt;= 6.0 : 超时改为 double 而不是 integer。

---

### Examples

```
redis> DEL list1 list2
(integer) 0
redis> RPUSH list1 a b c
(integer) 3
redis> BRPOP list1 list2 0
1) "list1"
2) "c"
```
