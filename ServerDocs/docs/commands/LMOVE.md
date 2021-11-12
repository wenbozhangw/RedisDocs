## LMOVE source destination LEFT|RIGHT LEFT|RIGHT

    起始版本: 6.2.0
    时间复杂度: O(1)

以原子方式返回并删除存储在 source 中的列表的第一个/最后一个元素（head/tail 取决于 wherefrom 参数），并将元素 push 到 destination 列表的第一个/最后一个元素（head/tail 取决于 whereto 参数）。

例如：假设 source 存储着列表 a、b、c，destination 存储着列表 x、y、z。执行 `LMOVE source distination RIGHT LEFT` 命令得到的结果是 source 保存着列表 a、b，而 destination 保存着列表 c、x、y、z。

如果 source 不存在，则返回 nil value 并不执行任何操作。如果 source 和 destination 相同，则该元素相当于从列表中删除第一个/最后一个元素并将其作为列表的第一个/最后一个元素推送，因此可以认为它是一个列表旋转命令（如果 wherefrom 和 whereto 相同，可认为是无操作）。

此命令代替现在已弃用的 [RPOPLPUSH](RPOPLPUSH.md)。作为 `LMOVE RIGHT LEFT` 是等效的。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 被移除和放入的元素。

---

### Examples

```
redis> RPUSH mylist "one"
(integer) 1
redis> RPUSH mylist "two"
(integer) 2
redis> RPUSH mylist "three"
(integer) 3
redis> LMOVE mylist myotherlist RIGHT LEFT
"three"
redis> LMOVE mylist myotherlist LEFT RIGHT
"one"
redis> LRANGE mylist 0 -1
1) "two"
redis> LRANGE myotherlist 0 -1
1) "three"
2) "one"
redis> 
```

---

### Pattern: Reliable queue

Redis 通常都被用做一个处理各种后台工作或消息任务的消息服务器。一个简单的队列模式就是：生产者把消息放入一个列表中，等待消息的消费者用 [RPOP](RPOP.md) 命令（用轮询方式），或者 [BRPOP](BRPOP.md) 命令（如果客户端使用阻塞操作会更好）来得到这个消息。

然而，因为消息有可能会丢失，所以这种队列并不是安全的。例如，当接收到消息后，出现了网络问题或者消费者端崩溃了，那么这个消息就丢失了。

[LMOVE](LMOVE.md)（或其阻塞版本的 [BLMOVE](BLMOVE.md)）提供了一种方法来避免这个问题：消费者端在取到消息的同时把该消息放入一个正在处理中的列表。当消息被处理之后，该命令会使用 [LREM](LREM.md) 命令来移除正在处理中列表中的对应消息。

另外，可以添加一个客户端来监控这个正在处理中的列表，如果有某些消息已经在这个列表中存在很长时间了（即超过一定地处理时限），那么客户端可以把这些超时消息重新加入到队列中。

---

### Pattern: Circular list

[LMOVE](LMOVE.md) 命令的 source 和 destination 是相同的时候，客户端在访问一个拥有 n 个元素的列表时，可以在 O(N) 时间里一个接一个获取列表元素，而不用像 [LRANGE](LRANGE.md) 那样需要把整个列表从服务器端传送到客户端。

上面这种模式即使在以下两种情况下依旧能很好地工作：
- 有多个客户端同时对同一个列表进行旋转，又从头开始这个操作。
- 有其他客户端在往列表末端加入新的元素。

这个模式让我们可以很容易地实现这样一个系统：有 N 个客户端，需要连续不断地对一批元素进行处理，而且处理的过程必须尽可能地快。一个典型的例子就是服务器上的监控程序：它们需要在尽可能短的时间内，并行地检查一批网站，确保它们的可访问性。

值得注意的是，使用这个模式的客户端是易于扩展(scalable)且安全的(reliable)，因为即使客户端把接收到的消息丢失了，这个消息依然存在于队列中，等下次迭代到它的时候，由其他客户端进行处理。