## RPOPLPUSH source destination

    起始版本: 1.2.0
    时间复杂度: O(1)

原子性地返回并移除存储在 source 的列表的最后一个元素（列表尾部元素），并把该元素放入存储在 destination 的列表的第一个元素位置（列表头部）。

例如：假设 source 存储着列表 a、b、c，destination 存储着列表 x、y、z。执行 [RPOPLPUSH](rpoplpush.md) 命令得到的结果是 source 保存着列表 a、b，而 destination 保存着列表 c、x、y、z。

如果 source 不存在，那么会返回 nil 值，并不会执行任何操作。如果 source 和 destination 是一样的，那么这个操作等同于移除列表最后一个元素并把该元素放在列表头部，所以这个命令也可以当做是一个旋转列表的命令。

根据 Redis 6.2.0，[RPOPLPUSH](rpoplpush.md) 被认为已弃用。请使用新代码中的 [BLMOVE](blmove.md)。

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings) : 被移除和放入的元素。

### Examples

```
redis> RPUSH mylist "one"
(integer) 1
redis> RPUSH mylist "two"
(integer) 2
redis> RPUSH mylist "three"
(integer) 3
redis> RPOPLPUSH mylist myotherlist
"three"
redis> LRANGE mylist 0 -1
1) "one"
2) "two"
redis> LRANGE myotherlist 0 -1
1) "three"
redis> 
```

### Pattern: Reliable queue

Redis 通常都被用做一个处理各种后台工作或消息任务的消息服务器。一个简单的队列模式就是：生产者把消息放入一个列表中，等待消息的消费者用 [RPOP](rpop.md) 命令（用轮询方式），或者 [BRPOP](brpop.md) 命令（如果客户端使用阻塞操作会更好）来得到这个消息。

然而，因为消息有可能会丢失，所以这种队列并不是安全的。例如，当接收到消息后，出现了网络问题或者消费者端崩溃了，那么这个消息就丢失了。

[RPOPLPUSH](rpoplpush.md)（或其阻塞版本的 [BRPOPLPUSH](brpoplpush.md)）提供了一种方法来避免这个问题：消费者端在取到消息的同时把该消息放入一个正在处理中的列表。当消息被处理之后，该命令会使用 [LREM](lrem.md) 命令来移除正在处理中列表中的对应消息。

另外，可以添加一个客户端来监控这个正在处理中的列表，如果有某些消息已经在这个列表中存在很长时间了（即超过一定的处理时限），那么客户端可以把这些超时消息重新加入到队列中。

### Pattern: Circular list

[RPOPLPUSH](rpoplpush.md) 命令的 source 和 destination 是 相同的时候，客户端在访问一个拥有 n 个元素的列表时，可以在 O(N) 时间里一个接一个获取列表元素，而不用像 [LRANGE](lrange.md) 那样需要把整个列表从服务器端传送到客户端。

上面这种模式即使在以下两种情况下依旧能很好地工作：
- 有多个客户端同时对同一个列表进行旋转，又从头开始这个操作。
- 有其他客户端在往列表末端加入新的元素。

这个模式让我们可以很容易地实现这样一个系统：有 N 个客户端，需要连续不断地对一批元素进行处理，而且处理的过程必须尽可能地快。一个典型的例子就是服务器上的监控程序：它们需要在尽可能短的时间内，并行地检查一批网站，确保它们的可访问性。

值得注意的是，使用这个模式的客户端是易于扩展(scalable)且安全的(reliable)，因为即使客户端把接收到的消息丢失了，这个消息依然存在于队列中，等下次迭代到它的时候，由其他客户端进行处理。