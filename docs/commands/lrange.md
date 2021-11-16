## LRANGE key start stop

    起始版本：1.0.0
    时间复杂度：O(S + N)，其中 S 是小列表从 HEAD 开始偏移的距离，对于大列表，从最近端（HEAD 或 TAIL）开始；N 是指定范围内的元素数量。

返回存储在 key 的列表里指定范围内的元素。start 和 end 偏移量都是基于 0 的下标，即 list 的第一个元素下标是 0（list的头部），第二个元素下标是 1，以此类推。

偏移量也可以是负数，表示偏移量是从 list 尾部开始计数。例如，-1 表示列表的最后一个元素，-2 是倒数第二个，以此类推。

---

### Consistency with range functions in various programming languages

需要注意的是，如果你有一个 list，里面的元素是从 0 到 100，那么 `LRANGE list 0 10` 这个命令会返回 11 个元素，即最右边的那个元素也会被包含在内。在你所使用的编程语言里，这一点可能是也可能不是跟哪些求范围有关的函数都是一致的。（像Ruby的 Range.new，Array#slice 或者Python的 range() 函数。）

---

### Out-of-range index

当下标超过 list 范围的时候不会产生 error。如果 start 比 list 的尾部下标大的时候，会返回一个空列表。如果 stop 比 list 的实际尾部大的时候，Redis会当它是最后一个元素下标。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) : 指定范围内的元素列表。

---

### Examples

```
redis> RPUSH mylist "one"
(integer) 1
redis> RPUSH mylist "two"
(integer) 2
redis> RPUSH mylist "three"
(integer) 3
redis> LRANGE mylist 0 0
1) "one"
redis> LRANGE mylist -3 2
1) "one"
2) "two"
3) "three"
redis> LRANGE mylist -100 100
1) "one"
2) "two"
3) "three"
redis> LRANGE mylist 5 10
(empty list or set)
redis> 
```