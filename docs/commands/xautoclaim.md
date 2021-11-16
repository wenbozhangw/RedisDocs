## XAUTOCLAIM key group consumer min-idle-time start [COUNT count] [JUSTID]

    起始版本：6.2.0.
    时间复杂度：O(1) 如果 COUNT 很小。

此命令转移与指定条件匹配的挂起流条目的所有权。从概念上讲，[XAUTOCLAIM](xautoclaim.md) 相当于先调用 [XPENDING](xpending.md)，然后调用 [XCLAIM](xclaim.md)，但提供了一种更直接的方法来通过类似 [SCAN](scan.md) 的语义处理消息传递失败。

与 [XCLAIM](xclaim.md) 一样，该命令在 `<key>` 和提供的 `<group>` 的上下文中对流条目进行操作。它将等待时间超过 `<min-idle-time>` 毫秒且 ID 等于或大于 `<start>` 的消息的所有权转移给 `<consumer>`。

可选的 `<count>` 参数默认为 100，是命令尝试认领的条目数的上限。在内部，该命令从 `<start>` 开始扫描消费者组的 Pending Entries List(PEL)，并过滤掉空闲时间小于或等于 `<min-idle-time>` 的条目。命令扫描的最大挂起条目数是 `<count>` 的值乘以 10 （硬编码）的乘积。因此，认领的条目数可能会少于指定值。

可选的 `JUSTID` 参数将恢复更改为仅返回成功认领的消息 ID 数组，而不返回实际消息。使用此选项意味着重试计数器不会增加。

该命令将认领的条目作为数组返回。它还返回一个 Stream ID，用于类似游标的用途，作为其后续调用的 `<start>` 参数。当没有剩余的 PEL 条目时，该命令返回特殊的 `O-O` ID 以表示完成。但是，请注意，即使在使用 `0-0` 作为 `<start>` ID 的扫描完成后，您可能仍希望继续调用 [XAUTOCLAIM](xautoclaim.md)，因为时间已过，因此较旧地待处理条目现在可能有资格进行认领。

请注意，只有空闲时间长于 `<min-idle-time>` 的消息才会被认领，并且认领消息会重置其空闲时间。这确保了只有一个消费者可以在特定的时刻成功认领给定的待处理消息，并且微不足道地降低了多次处理统一消息的可能性。

最后，使用 [XAUTOCLAIM](xautoclaim.md) 认领消息还会增加该消息的尝试传递计数，除非已指定 `JUSTID` 选项（仅传递消息 ID，而不传递消息本身）。由于某种原因无法处理的消息 —— 例如，因为使用者在处理它们时系统崩溃 —— 将显示出高的尝试传递次数，而监视可以检测到这些次数。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)，特定的：

包含两个元素的数组：

1. 第一个元素是一个 Stream ID，用作下一次调用 [XAUTOCLAIM](xautoclaim.md) 的 `<start>` 参数
2. 第二个元素是一个数组，其中包含与 [XRANGE](xrange.md) 格式相同的所有成功认领的消息。

---

### Examples

```
> XAUTOCLAIM mystream mygroup Alice 3600000 0-0 COUNT 25
1) "0-0"
2) 1) 1) "1609338752495-0"
      2) 1) "field"
         2) "value"
```

在上面的示例中，我们尝试认领多达 25 个挂起和空闲（未确认或认领）至少一个小时的条目，从 stream 的开头开始。来自 `mygroup` 组的消费者 "Alice" 获得了这些消息的所有权。请注意，示例中返回的 stream ID 为 0-0，表示扫描了整个 stream。