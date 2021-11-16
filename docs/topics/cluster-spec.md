## Redis Cluster Specification

欢迎使用 **Redis 集群规范**。在这里，您将找到有关 Redis 集群的算法和设计原理的信息。本文档是一项正在进行的工作，因为它与 Redis 的实际实现在不断同步。

---

## Main properties and rationales of the design

### Redis Cluster goals

Redis Cluster 是 Redis 的分布式实现，具有以下目标，按设计中的重要性排序：

- 高达 1000 个节点的高性能和线性可扩展性。没有代理，使用一步复制，并且不对值执行合并操作。
- 可接受的写入安全程度：那些与大多数节点相连的客户端所做的写入操作，系统尝试全部都保存下来。不过公认的，还是会有小部分（small windows）写入会丢失。
- 可用性(Availability)：Redis Cluster 能够在大多数主节点(master node)可访问的分区中存活下来，并对于每个不可达的主节点都至少有一个它的副本节点(replica)。此外，使用 *副本迁移(replicas migration)* ，不再被任何副本节点复制的主节点将从被多个副本节点覆盖的主节点接收一个。

本文档中描述的内容在 Redis 3.0 或更高版本中实现。

---

### Implemented subset

Redis Cluster 实现了非分布式版本 Redis 中可用的所有单 key 命令。执行复杂的多 key 操作的命令，如集合类型的并集(unions)或交集(intersections)，只要 key 都散列到同一个槽，就可以实现。

Redis Cluster 实现了一个称为 **hash tags** 的概念，可以使用它来强制某些 key 存储在同一个哈希槽中。但是在手动重新分片过程中，多 key 操作可能会在一段时间内变得不可用，而单 key 操作始终可用。


Redis Cluster 不支持像 Redis 的独立版本那样支持多个数据库。只有数据库 0 并且不允许使用 [SELECT](../commands/select.md) 命令。

---

### Clients and Servers roles in the Redis Cluster protocol

在 Redis Cluster 中，节点负责保存数据，并获取集群的状态，包括将 key 映射到正确的节点。集群节点还能够自动发现其他节点，检测非工作节点，并在需要时将副本节点提升为 master，以便在发生故障时继续运行。

为了执行这些任务，所有集群节点都使用 TCP 总线(TCP bus)和称为 **Redis Cluster Bus** 的二进制协议连接。每个节点都使用集群总线连接到集群中的每个其他节点。节点使用 gossip 协议来传播有关集群的信息以发现新节点、发送 ping 数据包以确保所有其他节点正常工作，以及在特定情况发生时发送集群消息。集群总线还用于在集群中传播 Pub/Sub 消息，并在用户请求时协调手动故障转移（手动故障转移不是由 Redis 集群故障检测器启动，而是由系统管理员直接启动的故障转移）。

由于集群节点无法代理请求，客户端可能会使用重定向错误 `-MOVED` 和 `-ASK` 重定向到其他节点。理论上来说，客户端可以自由地向集群中的所有节点发送请求，在需要的时候把请求重定向到其他节点，因此客户端不需要保存集群的状态。然而，能够缓存 key 和节点之间的映射的客户端可以以合理的方式提高性能。

---

### Write safety


Redis Cluster 使用节点间异步复制(asynchronous replication)，**最后一次故障转移赢得(last failover wins)** 隐式合并功能。这意味着最后选出的 master 数据集最终会替换所有其他副本节点。在分区过程中总是有可能丢失写入的时间窗口(window of time)。 但是一个连接到绝大部分主节点的客户端的时间段，与一个连接到极小部分主节点的客户端的时间段是相当不同的。

与在少数方执行的写入相比，Redis Cluster会努力尝试保存所有与大多数主节点连接的客户端执行的写入。以下是导致故障期间多数分区中收到的已确认写入丢失的场景示例：

1. 一个写入操作可能会达到主节点，但是虽然主节点可能能够回复客户端，但写入可能无法通过主节点和副本节点之间使用的异步复制，传播到副本节点。如果在某个写入没有到达副本节点的时候 master 宕机，并且 master 在足够长的时间内无法访问，导致副本节点提升为 master，则写入将永远丢失。在 master 突然完全故障的情况下，这通常很难观察到，因为主节点尝试在大约统一时间回复客户端（确认写入）和副本（传播写入）。然而，这是一种现实世界的故障模式。
2. 另一种理论上可能的写入丢失的故障模式如下：
   - 因为分区使 master 变得不可达。
   - 故障转移(fail over)到 master 的副本节点。（即副本节点提升为主节点）
   - 过一段时间之后 master 再次变得可达。
   - 一个没有更新路由表(routing table)的客户端或许会在集群把这个主节点变成一个副本节点（新主节点的副本节点）之前对它进行写入操作。

实际上这是极小概率事件，这是因为，那些由于长时间无法被大多数主节点访问到的节点，会被故障转移掉，不再接受任何写入操作，当其分区修复好以后仍然会在一小段时间内拒绝写入操作，以便让其他节点有时间被告知配置信息的变更。这种故障模式还要求客户的路由表尚未更新。

针对分区中少数部分的写操作有更大的丢失窗口。例如，Redis 集群在少数 master 和至少一个或多个客户端的分区上丢失了大量的写操作，因为如果 master 发生故障，所有发送给 master 的写操作都可能丢失。

具体来说，对要进行故障转移的 master，它必须至少在 `NODE_TIMEOUT` 内无法被大多数 master 访问，因为如果分区在这段时间之前已修复，则不会丢失任何写入。当分区故障持续超过 `NODE_TIMEOUT` 时，在少数方执行的所有写入操作可能会丢失。然而，Redis 集群的少数方会在 `NODE_TIMEOUT` 时间过去后立即开始拒绝写入，而不会与多数方联系，因此存在一个最大窗口，之后少数方将不再可用。因此，在那之后，不会接受或丢失任何写入。

---

### Availability

Redis Cluster 在分区的少数方不可用。在分区的多数方，假设至少有大多数 master 节点可达，并且对于每个不可达 master 都有至少一个副本节点可达，在经过了差不多 `NODE_TIMEOUT` 这么长时间后，有个副本节点会被推选出来并故障转移掉它的 master，这时集群又再次恢复可用。

这意味着 Redis Cluster 的设计是能容忍集群中少数节点的出错，但对于要求大量网络分块（large net splits）的可用性的应用来说，这并不是一个合适的解决方案。

举个自理，一个由 N 个 master 组成的集群，每个 master 都只有一个 replica。当有一个节点（因为故障）被分割出去后，集群的多数节点这边仍然是可访问的。当有两个节点（因为故障）被分割出去后，集群仍可用的概率是 `1-(1/(N*2-1))`（在第一个节点故障出错后总共剩下 `N*2-1` 个节点，那么是去副本节点的那个主机诶单也故障出错的概率是 `1/(N*2-1)`）。

比如一个拥有 5 个节点的集群，每个节点都只有一个副本节点，那么在两个节点从多数节点这边分割出去后，集群不再可用的概率是  1/(5*2-1) = 0.1111，即有大约 11% 的概率。

由于 Redis 集群功能称为 **副本迁移(replicas migration)**，通过将副本迁移到孤立的 master 节点（主机诶单不再具有副本）这一事实，集群可用性在许多实际场景中得到了提高。因此，在每个成功的故障故障事件中，集群可能会重新配置副本节点布局，以便更好地抵抗下一次故障。

---

### Performance

在 Redis 集群中，节点不会将命令代理到负责给定 key 的正确节点，而是将客户端重定向到为 keyspace 的给定部分提供服务的正确节点。

最终，客户端获得一份最新的集群表示，里面有写着哪些节点服务哪些 key 的子集提供服务，因此在正常操作期间，客户端直接联系正确的节点已发送给定的命令。

由于使用异步复制，节点不会等待其他节点对写入的确认（如果没有使用 [WAIT](../commands/wait.md) 命令明确请求）。

此外，由于多 key 命令仅限于 *相邻(near)* key，因此除非重新分片，否则数据永远不会在节点之间移动。

正常操作的处理与单个 Redis 实例的情况完全相同。这意味着在具有 N 个 master 的 Redis 集群中，随着设计的线性扩展，您可以获得与单个 Redis 实例乘以 N 相同的性能。同时，查询通常在一次往返中被执行，因为客户端通常保持与节点的持久连接，因此延迟数字也与单个独立的 Redis 节点情况相同。

非常高的性能和可扩展性，同时保留弱但合理的数据安全性和可用性形式是 Redis Cluster 的主要目标。

---

### Why merge operations are avoided

Redis Cluster 设计避免了多个节点中相同键值对的版本冲突，这是因为 Redis 数据模型并不提倡这么做。Redis 中的值通常非常大；通常会看到包含数百万个元素的列表或 sorted set。数据类型在语义上也很复杂。传输和合并这些类型的值可能是一个主要的性能瓶颈，或可能需要应用程序端非正常使用、额外的内存来存储元数据等。

这里没有严格的技术限制。CRDT 或同步复制状态机(synchronously replicated state machines)可以模拟类似于 Redis 的复杂数据类型。但是，此类系统的实际运行时行为与 Redis Cluster 不同。Redis Cluster 旨在涵盖非集群 Redis 版本的全部用例。

---

## Overview of Redis Cluster main components

### Keys distribution model

keyspace 被分成 16384 个槽(slot)，事实上集群的最大节点数量是 16384 个（然而建议最大节点数量大约在 1000 个左右）。

集群中的每个 master 都负责 16384 个哈希槽中的一部分。当集群处于稳定状态时，集群中没有在执行重配置(reconfiguration)操作。当集群稳定时，每个哈希槽都只有一个节点进行处理（不过主节点可以有一个或多个副本节点，可以在网络断线或节点失效时替换掉主节点）。

用于将 key 映射到哈希槽的算法如下（下一段 hash tag exception 就是按照这个规则）：

```
HASH_SLOT = CRC16(key) mod 16384
```

其中，CRC16 的定义如下：

- Name: `XMODEM`（也可以称为 `ZMODEM` 或 `CRC-16/ACORN`）
- Width: 16 bit
- Poly: 1021 (即 x<sup>16</sup> + x<sup>12</sup> + x<sup>5</sup> + 1)
- Initialization: 0000
- 反射输入字节(Reflect Input byte): False
- 反射输入CRC(Reflect Output CRC): False
- 用于输出CRC的异或常量(Xor constant to output CRC): 0000
- 该算法对 "123456789" 输出: 31C3

CRC16 的 16 位输出中的 14 位会被使用（这也是为什么上面的式子中有一个对 16384 取余的操作）。在我们的测试中，CRC16 能相当好地把不同的 key 均匀地分配到 16384 个槽中。

注意：在本文档的附录 A 中有 CRC16 算法的实现。

---

### Keys hash tags

计算哈希槽可以实现 **哈希标签(hash tags)**，但这有一个例外。哈希标签是确保两个 key 都在同一个哈希槽里的一种方式。将来也许会使用到哈希标签，例如为了在集群稳定的情况下（没有在做碎片重组操作）允许某些多 key 操作。

为了实现哈希标签，哈希槽是用另一种不同的方式计算。基本来说，如果一个 key 包含一个 "{...}" 这样的模式，只有 `{` 和 `}` 之间的字符串会被用来做哈希以获取哈希槽。但是有可能出现多个 `{` 或 `}`，计算的算法如下：

- 如果 key 包含一个 `{` 字符。
- 那么在 `{` 的右边就会有一个 `}`。
- 在 `{` 和 `}` 之间会有一个或多个字符，第一个 `}` 一定是出现在第一个 `{` 之后。

然后不是直接计算 key 的哈希，只有在第一个 `{` 和它右边第一个 `}` 之间的内容会被用来计算哈希值。

例子：

- 比如这两个 key `{user1000}.following` 和 `{user1000}.followers` 会被哈希到同一个哈希槽里，因为只有 `user1000` 这个子串会被用来计算哈希值。
- 对于 `foo{}{bar}` 这个 key，整个 key 都会被用来计算哈希值，因为第一个出现的 `{` 和它右边第一个 `}` 之间没有任何字符串。
- 对于 `foo{{bar}}zap` 这个 key，子串 `{bar` 将被用来计算哈希值，因为它是第一个 `{` 及其右边第一个 `}` 之间的内容。
- 对于 `foo{bar}{zap}` 这个 key，用来计算哈希值的是 `bar` 这个子串，因为算法会在第一次有效或无效（比如中间没有任何字节）地匹配到 `{` 和 `}` 的时候停止。
- 按照这个算法，如果一个 key 是以 `{}` 开头的话，那么就当做整个 key 都会被用来计算哈希值。当使用二进制数据作为 key 名称的时候，这是非常有用的。

下面是用 Ruby 和 C 语言实现的 HASH_SLOT 函数，Adding the hash tags exception。

Ruby example code :

```Ruby
def HASH_SLOT(key)
    s = key.index "{"
    if s
        e = key.index "}",s+1
        if e && e != s+1
            key = key[s+1..e-1]
        end
    end
    crc16(key) % 16384
end
```

C example code:

```C
unsigned int HASH_SLOT(char *key, int keylen) {
    int s, e; /* start-end indexes of { and } */

    /* Search the first occurrence of '{'. */
    for (s = 0; s < keylen; s++)
        if (key[s] == '{') break;

    /* No '{' ? Hash the whole key. This is the base case. */
    if (s == keylen) return crc16(key,keylen) & 16383;

    /* '{' found? Check if we have the corresponding '}'. */
    for (e = s+1; e < keylen; e++)
        if (key[e] == '}') break;

    /* No '}' or nothing between {} ? Hash the whole key. */
    if (e == keylen || e == s+1) return crc16(key,keylen) & 16383;

    /* If we are here there is both a { and a } on its right. Hash
     * what is in the middle between { and }. */
    return crc16(key+s+1,e-s-1) & 16383;
}
```

---

### Cluster nodes attributes

在集群中，每个节点都有一个唯一的名字。节点名字是一个十六进制表示的 160 bit 随机数，这个随机数是节点第一次启动时获得的（通常使用 /dev/urandom）。节点会把它的 ID 保存在配置文件里，以后永远使用这个 ID，只要这个节点配置文件没有被系统管理员删掉，或者通过 [CLUSTER RESET](../commands/cluster-reset.md) 命令请求 *硬重置(hard reset)* 。

节点 ID 是用于在整个集群中标识每个节点。一个给定的节点可以在不改变节点 ID 的情况下改变 IP 地址。集群能检测到 IP 或端口的变化，并使用运行在集群总线(cluster bus)上的 gossip 协议重新配置。

节点 ID 不是与每个节点相关联的唯一信息，而是唯一始终全局一致的信息。每个节点还具有以下相关联的信息集合。一些信息是关于这个特定节点的集群配置细节的，并且最终在整个集群中是一致的。其他一些信息，例如上次 ping 节点的时间，则是每个节点的本地信息。

每个节点维护以下关于它在集群中知道的其他节点的信息：
- 节点 ID
- 节点 IP 和端口
- 一个各种 flags 的集合
- 如果它是一个副本节点，则该节点的主节点是什么
- 最后一次节点被 ping 的时间
- 最后一次收到 pong 的时间
- 节点的当前 _configuration epoch_ （在本规范后面解释）
- 连接状态
- 节点使用的哈希槽集合。


[CLUSTER NODES](../commands/cluster-nodes.md) 文档中描述了 [explanation of all the node fields](../commands/cluster-nodes.md) 的详细说明。

[CLUSTER NODES](../commands/cluster-nodes.md) 命令可以发送到集群中的任意节点，根据被查询节点对集群的本地视图，提供集群的状态和每个节点的信息。

以下是发送到由三个节点组成的小型集群中的主节点的 [CLUSTER NODE](../commands/cluster-nodes.md) 命令的示例输出：

```
$ redis-cli cluster nodes
d1861060fe6a534d42d8a19aeb36600e18785e04 127.0.0.1:6379 myself - 0 1318428930 1 connected 0-1364
3886e65cc906bfd9b1f7e7bde468726a052d1dae 127.0.0.1:6380 master - 1318428930 1318428931 2 connected 1365-2729
d289c575dcbc4bdd2931585fd4339089e461a27d 127.0.0.1:6381 master - 1318428931 1318428931 3 connected 2730-4095
```

在上面的列表中，不同的字段按顺序排列：node id、address:port、flags、last ping sent、last pong received、configuration epoch、link state、slots。一旦我们谈到 Redis Cluster 的特定部分，就会涉及到有关上述字段的详细信息。

---

### The Cluster bus

每个 Redis Cluster 节点都有一个额外的 TCP 端口，用于接收来自其他 Redis 集群节点的传入连接。该端口通过将 10000 添加到数据端口来派生，或者可以使用 cluster-port 配置指定。

示例 1：

如果 Redis 节点正在监听 6379 端口的客户端连接，并且你没有在 redis.conf 中添加 cluster-port 参数，则会打开 Cluster bus 的 16379 端口。

示例 2：

如果 Redis 节点在 6379 端口监听客户端连接，并且你在 redis.conf 中设置了 cluster-port 20000，那么 Cluster bus 的 20000 端口就会被打开。

节点到节点的通信只使用集群总线和集群总线协议：一种由不同类型和大小的 frames 组成的二进制协议。集群总线二进制协议没有公开的文档，因为他不适合外部软件设备使用该协议与 Redis 集群节点通信。但是，您可以通过阅读 Redis 集群源代码中的 cluster.h 和 cluster.c 文件来获取有关集群总线协议的更多详细信息。

---

### Cluster topology

Redis Cluster 是一个网状(mesh)结构，其中每个节点都使用 TCP 连接与其他每个节点相连。

在 N 个节点的集群中，每个节点都有 N-1 个传出 TCP 连接和 N-1 个传入连接。

这些 TCP 连接始终保持活动状态，不会按需创建。当一个节点期望 pong 回复以响应集群总线中的 ping 时，在等待足够长的时间以将该节点标记为不可达之前，它将尝试通过从头开始重新连接来刷新与该节点的连接。

而 Redis Cluster 节点形成全网状(full mesh)，**节点使用 gossip 协议和配置更新机制，以避免在正常情况下节点之间交换过多的消息，** 因此交换的消息数量不是指数级的。

---

### Node handshake

节点总是在集群连接断开接受连接，甚至会回复接收到的 ping 包，即使发送 ping 包的节点是不可信的。然而如果某个节点不被认为是在集群中的，那么所有它发出的数据包都会被丢弃掉。

只有在两种方式下，一个节点才会认为另一个节点是集群中的一部分：

- 如果节点使用 MEET 消息介绍自己。一个 meet 消息与 [PING](../commands/ping.md) 消息完全一样，但它会强制让接受者接受发送者为集群中的一部分。只有在系统管理员使用以下命令要求的时候，节点才会发送 MEET 消息给其他节点：
```
CLUSTER MEET ip port
```
- 一个已被信任的节点能通过传播gossip消息，让另一个节点被注册此节点，作为集群中的一部分。也就是说，如果 A 知道 B ，B 知道 C，那么 B 会向 A 发送 C 的 gossip 消息。A 收到后就会把 C 当做是网络中的一部分，并且尝试连接 C。

这意味着，只要我们往任何连接图(connected graph)中接入节点，它们最终会自动形成一个完全连接图。从根本上来说，这表示集群能自动发现其他节点，但前提是有一个由系统管理员强制创建的信任关系。

这个机制能防止不同的 Redis 集群因为 IP 地址的变更或者其他网络事件而意外混合起来，从而使集群更具健壮性。

---

## Redirection and resharding

### MOVED Redirection

一个 Redis 客户端可以自由地向集群中的任意节点（包括从节点）发送查询。接收的节点会分析查询，如果这个命令是集群可以执行的（就是查询中只涉及一个 key），那么节点会找到这个 key 所属的哈希槽对应的节点。

如果刚好这个节点就是对应这个哈希槽，那么这个查询就直接被节点处理掉。否则这个节点会查看它内部的哈希槽到节点映射，然后给客户端返回一个 MOVED 错误。 一个 MOVED 错误如下：

```
GET x
-MOVED 3999 127.0.0.1:6381
```

这个错误包括 key 对应的哈希槽（3999）和能处理这个查询的节点的 ip:port（127.0.0.1:6381）。客户端需要重新发送查询到给定的 ip 地址和端口号的节点。注意，即使客户端在重发查询之前等待了很长一段时间，一次同时集群的配置信息发生改变，如果哈希槽 3999 现在是为其他节点服务，那么目标节点会再向客户端回复一个 MOVED 错误。

从集群的角度看，节点是以 ID 来标识的。我们尝试简化接口，所以只向客户端暴露哈希槽和用 "ip:port" 标识的 Redis 节点之间的映射。

虽然并没有要求，但是客户端应该尝试记住哈希槽 3999 服务于 127.0.0.1:6381。这样的话一旦有一个新的命令需要发送，它能计算出目标 key 的哈希槽，提高找到正确节点的几率。

另一种方法是在收到 MOVED 重定向时使用 [CLUSTER NODES](../commands/cluster-nodes.md) 或 [CLUSTER SLOTS](../commands/cluster-slots.md) 命令刷新整个客户端集群布局。当遇到重定向时，可能会重新配置多个哈希槽而不是一个，因此尽快更新客户端配置通常是最佳策略。

注意，当集群稳定的时候，所有客户端最终都会得到一份哈希槽到节点的映射表，这样能使得集群效率非常高：客户端直接定位目标节点，不用重定向、或代理或发生其他单点故障(single point of failure entities)。

客户端还 **必须能够处理本文档后面描述的 -ASK 重定向**，否则他不是完整的 Redis Cluster 客户端。

---

### Cluster live reconfiguration

Redis Cluster 支持在集群运行过程中添加或移除节点。实际上，添加或移除节点都被抽象为同一个操作，那就是把哈希槽从一个节点移动到另一个节点。这意味着可以使用相同的基本机制(basic mechanism)来重新平衡集群、添加或删除节点等。

- 向集群添加一个新节点，就是把一个空节点加入到集群中并把某些哈希槽从已存在的节点迁移到新节点上。
- 从集群中移除一个节点，就是把该节点上的哈希槽迁移到其他已经存在的节点上。
- 为了重新平衡集群，在节点之间移动一组给定的哈希槽。

所以实现这个的核心是能把哈希槽移来移去。从实际角度看，哈希槽只是一组 key，因此 Redis Cluster 在重新分片期间真正做的是将 key 从一个实例移动到另一个实例。移动一个哈希槽意味着将所有碰巧哈希后对应这个槽的 key 移动到这个哈希槽中。

为了理解其工作原理，我们需要展示用于操作 Redis Cluster 节点上的哈希槽转换表(slots translation table) 的 CLUSTER 子命令。

以下是可用的子命令：

- [CLUSTER ADDSLOTS](../commands/cluster-addslots.md) slot1 [slot2] ... [slotN]
- [CLUSTER DELSLOTS](../commands/cluster-delslots.md) slot1 [slot2] ... [slotN]
- [CLUSTER SETSLOT](../commands/cluster-setslot.md) slot NODE node
- [CLUSTER SETSLOT](../commands/cluster-setslot.md) slot MIGRATING node
- [CLUSTER SETSLOT](../commands/cluster-setslot.md) slot IMPORTING node

前两个命令 `ADDSLOTS` 和 `DELSLOTS` 仅用于给一个 Redis 节点指派(assign)或移除哈希槽。分配槽意味着告诉给定的主节点它将负责为指定的哈希槽存储和提供内容。

分配哈希槽后，它们将使用 gossip 协议在集群中传播，如后面 _configuration propagation_ 部分所述。

`ADDSLOTS` 命令通常用于从头创建新集群时，为每个主节点分配所有 16384 个可用哈希槽的子集。

`DELSLOTS` 主要用于手动修改集群配置或调试任务：在实践中很少使用。

如果使用 `SETSLOT <slot> NODE` 形式，则 `SETSLOT` 子命令用于将槽分配给特定节点 ID。否则，槽可以设置为两种特殊状态 `MIGRATING` 和 
`IMPORTING`。这两种特殊状态用于将哈希槽从一个节点迁移到另一个节点。

- 当一个槽被设置为 `MIGRATING`，原来持有该哈希槽的节点仍会接受所有跟这个哈希槽有关的请求，但只有当查询的 key 还存在原节点时，原节点会处理该请求，否则这个查询会通过一个 -ASK重定向(-ASK 
  redirection) 转发到迁移的目标节点。
- 当一个槽被设置为 `IMPORTING`，只有在接收到 [ASKING](../commands/asking.md) 命令之后节点才会接收所有查询这个哈希槽的请求。如果客户端一直没有发送 [ASKING](../commands/asking.md) 命令，那么查询都会通过 -MOVE重定向(-MOVE redirection) 错误信息转发到真正处理这个哈希槽的节点那里。

那么讲可能显得有点奇怪，现在我们用实例让它更清晰些。假设我们有两个 Redis 节点，称为 A 和 B。我们想要把哈希槽 8 从节点 A 转移到节点 B，所以我们发送了这样的命令：

- 我们向节点 B 发送：`CLUSTER SETSLOT 8 IMPORTING A`
- 我们向节点 A 发送：`CLUSTER SETSLOT 8 MIGRATING B`

其它所有节点在每次被询问到的一个 key 是属于哈希槽 8 的时候，都会把客户端引向节点 A。具体如下：

- 所有关于已存在的 key 的查询都由节点 A 处理。
- 所有关于不存在于节点 A 的 key 都由节点 B 处理。

这种方式让我们可以不用在节点 A 中创建新的 key。同时，在重新分片和 Redis Cluster 配置期间使用的 redis-cli 会将哈希槽 8 中现有的 key 从 A 迁移到 B。这是使用以下命令执行的：

```
CLUSTER GETKEYSINSLOT slot count
```

上面这个命令会返回指定的哈希槽中的 count 个 key。对于每个返回的 key，redis-cli 向节点 A 发送一个 [MIGRATE](../commands/migrate.md) 命令，该命令以原子方式将指定的 key 从 A 迁移到 B（在移动 key 的过程中，两个节点都被锁住（通常是非常短的时间），以避免出现竞争状况）。以下是 [MIGRATE](../commands/migrate.md) 的工作原理：

```
MIGRATE target_host target_port "" target_database id timeout KEYS key1 key2 ...
```

执行 [MIGRATE](../commands/migrate.md) 命令的节点会连接到目标节点，把序列化后的 key 发送过去，一旦接收到 OK 回复就会从它自己的数据集中删除旧的 key。从外部客户端的角度来看，key 在任何给定时间都存在于 A 或 B 中。

在 Redis Cluster 中不需要指定 0 以外的数据库，但是 [MIGRATE](../commands/migrate.md) 是一个通用命令，可以用于其他不涉及 Redis Cluster 的任务。[MIGRATE](../commands/migrate.md) 经过优化，即使在移动复杂的 key（如长列表）时也能尽可能快地运行，但在 Redis Cluster 中，如果使用数据库的应用程序存在延迟，则在存在 big key 的情况下重新配置集群是很不明智的。

当迁移过程最终完成的时候，`SETSLOT <slot> NODE <node-id>` 命令被发送到迁移中涉及的两个节点，以便再次将槽者视为它们的正常状态。通常将相同的命令发送到所有其他节点，以免等待新配置在集群中自然传播。

---

### ASK redirection 

