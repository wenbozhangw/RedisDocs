## WAIT numslaves timeout

    起始版本：3.0.0
    时间复杂度：O(1)。

此命令阻塞当前客户端，直到所有以前的写入命令都成功的传输和指定的 slaves 确认。如果达到以毫秒为单位指定的超时，即使指定的 slaves 还没有到达，命令仍然返回。

[WAIT](wait.md) 命令**始终返回**之前写命令发送的 slaves 的数量，无论是在指定 slaves 的情况还是达到超时。

注意点：

1. 当 [WAIT](wait.md) 返回时，所有之前的写明了保证接收由 [WAIT](wait.md) 返回的 slaves 的数量。
2. 如果命令被作为 [MULTI](MULTI.md) 事务的一部分发送，该命令不阻塞，而是只尽快返回先前写命令的 slaves 的数量。
3. 如果 timeout 是 0 那意味着永远阻塞。
4. 由于 [WAIT](wait.md) 返回的是在失败和成功的情况下的 slaves 的数量。客户端应检查返回的 slaves 的数量是等于或更大的复制水平。

### Consistency and WAIT

[WAIT](wait.md) 不能保证 Redis 强一致性：尽管同步复制是复制状态机的一部分，但是还需要其他条件。不过，在 sentinel 和 Redis 集群故障转移中， [WAIT](wait.md) 能够增强数据的安全性。

如果写操作已经被传送给一个或多个 slave 节点，当 master 发生故障，我们极大概率（不保证 100%）提升一个收到写命令的 slave 节点为 master：不管是 Sentinel 还是 Redis Cluster 都会尝试选 slave 节点中最优（日志最新）的节点，提升为 master。尽管是选择最优节点，但是仍然会有丢失一个同步写操作可能性。

### Implementation details

因为引入了部分同步，Redis slave 节点在 ping 主节点时会携带已经处理的复制偏移量。这被用在多个地方：

1. 检测超时的 slaves
2. 断开连接后的部分复制
3. 实现 [WAIT](wait.md)

在 [WAIT](wait.md) 实现的案例中，当客户端执行完一个写命令后，针对每一个复制客户端，Redis会为其记录写命令产生的复制偏移量。当执行命令 [WAIT](wait.md) 时，Redis 会检测 slaves 节点是否已确认完成该操作或更新的操作。

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) : 当前连接的写操作会产生日志偏移，该命令会返回已处理至偏移量的 slaves 的个数。

### Examples

```
> SET foo bar
OK
> WAIT 1 0
(integer) 1
> WAIT 2 1000
(integer) 1
```

在例子中，第一次调用 [WAIT](wait.md) 并没有使用超时设置，并且设置写命令传输到一个 slaves 节点，返回成功。第二次使用时，我们设置了超时值并要求写命令传输到两个节点。因为只有一个 slaves 节点有效，1秒后 [WAIT](wait.md) 解除阻塞并返回 1 —— 传输成功的 slave 节点数量。
