## Data types

### Strings

String 是一种最基本的 Redis 值类型。Redis String 是二进制安全的，这意味着一个 Redis String 能包含任意类型的数据，例如：一张 JPEG 图片或者一个序列化的 Ruby 对象。

一个 String 类型的值最多能存储 512M 字节的内容。

你可以用 Redis String 做许多有趣的事，例如你可以：
- 利用 INCR 命令族（[INCR](../commands/incr.md)、[DECR](../commands/decr.md)、[INCRBY](../commands/incrby.md)）来把 String 当做原子计数器使用。
- 使用 [APPEND](../commands/append.md) 命令在 String 后添加内容。
- 将 String 作为 [GETRANGE](../commands/getrange.md) 和 [SETRANGE](../commands/setrange.md) 的随机访问向量。
- 在小空间里编码大量数据，或者使用 [GETBIT](../commands/gitbit.md) 和 [SETBIT](../commands/setbit.md) 创建一个 Redis 支持的 Bloom 过滤器。

查看所有 [available string commands](https://redis.io/commands/#string) 以获取更多信息，或阅读 [introduction to Redis data types](data-types-intro.md) 。

---

### Lists

Redis List 是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到 List 的头部（左边）或者尾部（右边）。

[LPUSH](../commands/lpush.md) 命令插入一个新元素到 List 头部，而 [RPUSH](../commands/rpush.md) 命令插入一个新元素到 List 尾部。当对一个空 key 执行其中某个命令时，将会创建一个新表。类似的，如果一个操作要清空 List ，那么 key 会从对应的 keyspace 删除。这是非常方便的语义，因为如果使用一个不存在的 key 作为参数，所有的 List 命令都会相对一个空表操作一样。

一些 List 操作及其结果：
```
LPUSH mylist a   # now the list is "a"
LPUSH mylist b   # now the list is "b","a"
RPUSH mylist c   # now the list is "b","a","c" (RPUSH was used this time)
```

List 的最大长度为 2<sup>32</sup> - 1 个元素(4294967295，每个 List 超过 40 亿个元素)。


从时间复杂度的角度来看，Redis Lists 的主要特点是支撑常量时间插入和删除靠近头部和尾部的元素，即使插入了数百万个条目。访问 List 两端的元素是非常快的，但如果你试着访问一个非常大的 List 的中间元素，则速度很慢，因为它是 O(N) 操作。

你可以用 Redis List 做许多有趣的事，例如你可以：
- 在社交网络中建立一个时间线模型，使用 [LPUSH](../commands/lpush.md) 去添加新的元素到用户时间线中，使用 [LRANGE](../commands/lrange.md) 去检索一些最近插入的条目。
- 你可以同时使用 [LPUSH](../commands/lpush.md) 和 [LTRIM](../commands/ltrim.md) 去创建一个永远不会超过指定元素数目的 List ，并同时记住最后的 N 个元素。
- List 可以用来当做消息传递的基元(primitive)，例如，众所周知的用来创建后台任务的 [Resque](https://github.com/resque/resque) Ruby 库。
- 你可以使用 List 做更多事，这个数据类型支持许多命令，包括像 [BLPOP](../commands/blpop.md) 这样的阻塞命令。请查看所有可用的 List 操作命令获取更多的信息。

查看完整的 [available commands operating on lists](https://redis.io/commands#list) 获取更多的信息，或进一步阅读 [introduction to Redis data types](data-types-intro.md) 。

---

### Sets

Redis Sets 是一个无序的字符串合集。你可以以 **O(1)** 的时间复杂度（无论集合中有多少元素时间复杂度都为常量）完成添加、删除和测试成员是否存在的操作。

Redis Sets 具有不允许重复成员的优秀特性。向 Set 中多次添加同一个元素，在集合中最终只会存在一个此元素。实际上这就意味着，在添加元素前，你并不需要实现进行检验此元素是否已经存在的操作。

一个 Redis List 十分有趣的是，它们支持一些服务端的命令从现有的 Set 出发去进行 Set 运算。所以你可以在很短的时间内完成合并(union)，交集(intersection)，找出不同元素的操作。

一个集合最多可以包含 2<sup>32</sup> - 1 个元素（4294967295，每个集合超过40亿个元素）。

你可以用 Redis Set 做很多有趣的事，例如你可以：
- 用 Set 跟踪一个独特的事。想要知道所有访问某个博客文件的独立 IP？只要每次都用 [SADD](../commands/sadd.md) 来处理一个页面访问。那么你可以肯定重复的 IP 是不会插入的。
- Redis Set 能很好地表示关系。你可以创建一个 tagging 系统，然后用 Set 来代表单个 tag。接下来你可以用 [SADD](../commands/sadd.md) 命令把所有拥有 tag 的对象的所有 ID 添加进 Set，这样来表示这个特定的 tag。如果你想要同时有 3 个不同的 tag 的所有对象的所有 ID，那么你需要使用 [SINTER](../commands/sinter.md)。
- 使用 [SPOP](../commands/spop.md) 或者 [SRANDMEMBER](../commands/srandmember.md) 命令随机地获取元素。

查看 [full list of Set commands](https://redis.io/commands#set) 获取更多信息，或者进一步阅读 [introduction to Redis data types](data-types-intro.md)。

---

### Hashes

Redis Hashes 是字符串 field 和字符串 value 之间的映射，所以它们可以完美地表示对象（eg：一个有名，姓，年龄等属性的用户）的数据类型。

```
HMSET user:1000 username antirez password P1pp0 age 34
HGETALL user:1000
HSET user:1000 password 12345
HGETALL user:1000
```

一个拥有少量（100个左右）字段的 hash 需要很少的空间来存储，所以你可以在一个小型的 Redis 实例中存储上百万的对象。

尽管 Hashes 主要用来表示对象，但它们也能够存储许多元素，所以你也可以用 Hashes 来完成许多其他的任务。

每个 hash 最多可以包含 2<sup>32</sup> - 1 个 key-value pair （超过 40 亿）。

查看 [full list of Hash commands](https://redis.io/commands#hash) 获取更多信息，或者进一步阅读 [introduction to Redis data types](data-types-intro.md)。

---

### Sorted sets

Redis Sorted Sets 和 Redis Sets 类似，都是不包含相同字符串的合集。它们的差别是，每个 Sorted Set 的每个成员都关联着一个评分，这个评分用于把 Sorted Set 中的成员按最低分到最高分排列。

使用 Sorted Set，你可以非常快地在 **O(log(N))** 完成添加，删除和更新元素的操作。因为元素是在插入时就排好序的，所以很快地通过评分（score）或者位置（position）获得一个范围元素。访问 Sorted Set 的中间元素同样也是非常快的，因此你可以使用 Sorted Set 作为一个没有重复成员的智能列表。在这个列表中，你可以轻易地访问任何你需要的元素：有序的元素，快速的存在性测试，快速访问集合中间元素！

简而言之，使用 Sorted Set 你可以很好地完成很多在其他数据库中难以实现的任务。

使用 Sorted Set 你可以：
- 在一个巨型在线游戏中建立一个排行榜，每当有新地记录产生时，使用 [ZADD](../commands/zadd.md) 来更新它。你可以用 [ZRANGE](../commands/zrange.md) 轻松地获取排名靠前的用户，你也可以提供一个用户名，然后用 [ZRANK](../commands/zrank.md) 获取他在排行榜中的名次。同时使用 [ZRANK](../commands/zrank.md) 和 [ZRANGE](../commands/zrange.md) 你可以获得与指定用户具有相同分数的用户名单。所有这些操作都非常迅速。
- Sorted Set 通常用来索引存储在 Redis 中的数据。例如：如果你有很多的 hash 来表示用户，那么你可以使用一个 Sorted Set，这个集合的年龄字段用来当做评分，用户 ID 当做值。用 [ZRANGEBYSCORE](../commands/zrangebyscore.md) 可以简单快速地检索到给定年龄段的所有用户。

Sorted Set 或许是最高级的 Redis 数据类型，所以或写时间查看 [full list of Sorted Set commands](https://redis.io/commands#sorted_set) 去探索你能用 Redis 干些什么吧！你也可以进一步阅读 [introduction to Redis data types](data-types-intro.md)。

---

### Bitmaps and HyperLogLogs

Redis 同样支持 Bitmaps 和 HyperLogLogs 数据类型，实际上是基于字符串的基本类型的数据类型，但有自己的语义。

请查看 [introduction to Redis data types](data-types-intro.md) 获取更多信息。