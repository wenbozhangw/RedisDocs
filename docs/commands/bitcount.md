## BITCOUNT key [start end]

    起始版本：2.6.0.
    时间复杂度：O(N)。

统计字符串被设置为 1 的 bit 数量。

一般情况下，给定的整个字符串都会被进行计数，通过指定额外的 start 和 end 参数，可以让计数只在特定的 bit 范围内进行。

start 和 end 参数的设置和 [GETRANGE](getrange.md) 命令类似，都可以使用负数值：比如 -1 表示最后一个 bit，而 -2 表示倒数第二个 bit，以此类推。

不存在的 key 被当成是空字符串来处理，因此对一个不存在的 key 进行 BITCOUNT 操作，结果为 0。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)

返回被设置为 1 的 bit 的数量。

---

### Examples

```
redis> SET mykey "foobar"
"OK"
redis> BITCOUNT mykey
(integer) 26
redis> BITCOUNT mykey 0 0
(integer) 4
redis> BITCOUNT mykey 1 1
(integer) 6
redis> 
```

---

### Pattern: real-time metrics using bitmaps

bitmap 对于一些特定类型的计算非常有效。一个例子是一个需要用户访问历史的 Web 应用程序，例如，它可以确定哪些用户是 beta 功能的良好目标。

使用 [SETBIT](setbit.md) 命令实现这一点很简单，用一个小的渐进整数标识每一天。例如， 0 是应用程序上线的第一天，1 是第二天，以此类推。

每次用户执行页面查看时，应用程序都可以使用设置域当天对应的 bit 的 [SETBIT](setbit.md) 命令来注册用户在当天访问了网站。

稍后，只需要针对 bitmap 调用 [BITCOUNT](bitcount.md) 命令即可知道用户访问该网站的单日天数。

使用用户 ID 而不是天数的类似模式在名为“ [Fast easy realtime metrics using Redis bitmaps](http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps) ”一文中有所描述。

---

### Performance considerations

前面的上线次数统计例子，即使运行 10 年，占用的空间也只是每个用户 10*365 bit，也即是每个用户 456 个字节。对于这种大小的数据来说，[BITCOUNT](bitcount.md) 的处理速度就像 [GET](get.md) 和 [INCR](incr.md) 这种 O(1) 复杂度的操作一样快。

如果你的 bitmap 数据非常大，那么可以考虑使用以下两种方法：

- 将一个大的 bitmap 分散到不同的 key 中，作为小的 bitmap 来处理。使用 Lua 脚本可以很方便地完成这一工作。
- 使用 [BITCOUNT](bitcount.md) 的 start 和 end 参数，每次只对所需的部分 bit 进行计算，将 bit 的累计工作放到客户端进行，并且对结果进行缓存。
