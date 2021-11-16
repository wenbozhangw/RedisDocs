## TIME

    起始版本：2.6.0。
    时间复杂度：O(1)。

[TIME](time.md) 命令用两个 item 的列表返回当前服务器的时间：Unix 时间戳 和 当前秒内已过去的微秒数。基本上，该接口与 gettimeofday 系统调用之一非常相似。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：

包含两个元素的 multi bulk reply：
- unix time in seconds;.
- microseconds.

---

### Examples

```
redis> TIME
1) "1632729443"
2) "770669"
redis> TIME
1) "1632729443"
2) "771074"
redis> 
```