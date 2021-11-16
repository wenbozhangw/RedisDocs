## EXPIRE key seconds [NX|XX|GT|LT]

    起始版本：1.0.0。
    时间复杂度：O(1)。

设置 key 的过期时间。超过过期时后，将会自动删除该 key。在 Redis 术语中，具有关联超时的 key 通常被称为易变的(volatile)。

超时后只对 key 执行 [DEL](del.md) 、 [SET](set.md) 、[GETSET](getset.md) 命令时才会被清除。这意味着，从概念上将所有改变 key 值的操作都会使他清除。例如，[INCR](incr.md) 递增 key 的值，执行 [LPUSH](lpush.md) 操作，或者用 [HSET](hset.md) 改变 hash 的字段，所有这些操作都会触发删除动作。

使用 [PERSIST](persist.md) 命令可以清除超时，使其变成一个永久的 key。

如果 key 被 [RENAME](rename.md) 命令修改，相关的超时时间会转移到新 key 上面。

如果 key 被 [RENAME](rename.md) 命令修改，比如原来就存在 Key_A，然后调用 `RENAME Key_B Key_A` 命令，这是不管原来 Key_A 是永久还是设置为超时，都会被 Key_B 的有效期状态覆盖。

注意，使用负数超时时间调用 [EXPIRE](expire.md)/[PEXPIRE](pexpire.md) 或使用过去时间调用 [EXPIREAT](expireat.md)/[PEXPIREAT](pexpireat.md) 将导致 key 被 [deleted](del.md) 而不是过期（因此，发出的 [key event](../topics/notifications.md) 将是删除，而不是过期）。

---

### Options

自 Redis 7.0 起，[EXPIRE](expire.md) 命令支持一组选项：
- NX —— 仅当 key 没有过期时间时，才设置过期时间。
- XX —— 仅当 key 存在过期时间时，才设置过期时间。
- GT —— 仅当新的过期时间大于当前过期时间时，才设置过期时间。
- LT —— 仅当新的过期时间小于当前过期时间时，才设置过期时间。

处于 GT 和 LT 的目的，非易变的(non-volatile) key 被视为无限的 TTL。GT、LT 和 NX 选项是互斥的。

---

### Refreshing expires

对已经有过期时间的 key 执行 [EXPIRE](expire.md) 操作，将会刷新它的过期时间。有很多应用有这种场景，例如记录会话的 session，下面的 *Navigation session*记录了一个实例。

---

### Differences in Redis prior 2.1.3

在 2.1.3 之前的 Redis 版本中，使用更改其值的命令（例如，set 命令）更改具有过期时间的 key ，具有完全删除 key 的效果。由于现在已修复的副本中的限制，因此需要此语义。

[EXPIRE](expire.md) 将返回 0 并且不会个更改设置了超时时间的 key 的 TTL。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特别的 ：
- 1 如果设置了超时。
- 0 如果key不存在或者不能设置过期时间。

---

### Examples

```
redis> SET mykey "Hello"
"OK"
redis> EXPIRE mykey 10
(integer) 1
redis> TTL mykey
(integer) 10
redis> SET mykey "Hello World"
"OK"
redis> TTL mykey
(integer) -1
redis> EXPIRE mykey 10 XX
ERR ERR wrong number of arguments for 'expire' command
redis> TTL mykey
(integer) -1
redis> EXPIRE mykey 10 NX
ERR ERR wrong number of arguments for 'expire' command
redis> TTL mykey
(integer) -1
redis> 
```

---

### History

- &gt;= 7.0：增加选项：NX, XX, GT 和 LT。

---

### Pattern: Navigation session

想象一下，你有一个网络服务器，你对用户最近访问的 N 个网页感兴趣，每个相邻的页面设置超时时间为 60 秒。在概念上，你为这些网页添加 Navigation session，如果你的用户，可能包含有趣的信息，他或她在寻找什么样的产品，你可以推荐相关产品。

你可以使用以下的策略模型，使用这种模式：每次用户浏览网页调用下面的命令：

```
MULTI
RPUSH pagewviews.user:<userid> http://.....
EXPIRE pagewviews.user:<userid> 60
EXEC
```

如果用户 60 秒没有操作，这个 key 将会被删除，不到 60 秒的话，后续网页将会被继续记录。

这个案例很容易用 [INCR](incr.md) 代替 [RPUSH](rpush.md)。

---

## Appendix : Redis expires

### Keys with an expires

通常 Redis key 创建时没有设置相关过期时间。它们会一直存在，除非使用显式的命令移除，例如，使用 [DEL](del.md) 命令。

[EXPIRE](expire.md) 一类命令能关联到一个有额外内存开销的 key。当 key 执行过期操作时，Redis 会确保按照规定的时间删除他们。

key 的过期时间和永久有效性可以通过 [EXPIRE](expire.md) 和 [PERSIST](persist.md) 命令（或者其他相关命令）来进行更新或者删除过期实现。

---

### Expire accuracy

在 Redis 2.4 及之前的版本，过期时间可能不是十分准确，有 0-1 秒的误差。

从 Redis 2.6 起，过期时间误差缩小到 0-1 毫秒。

---

### Expires and persistence

Key 的过期时间使用 Unix 时间戳存储（从 Redis 2.6 开始以毫秒为单位）。这意味着即使 Redis 实例不可用，时间也是一直在流逝的。

要想过期的工作处理好，计算机必须采用稳定的时间。如果你将 RDB 文件在两台时钟不同步的电脑间同步，有趣的事会发生（所有的 keys 装载时就会过期）。

即使在运行的实例也会检查计算机的时钟，例如你设置了一个 key 的有效期是 1000 秒，然后设置你的计算机为未来 2000 秒，这是 key 会立即失效，而不是等 1000 秒之后。

---

### How Redis expires keys

Redis key 过期有两种方式：被动和主动。

当一些客户端尝试访问它的时候，key会发送并主动的过期。

当然，这样是不够的，因为有些过期的 key，永远不会访问它们。无论如何，这些 key 应当过期，因此 Redis 会定时在具有过期时间的 key 中随机测试一些。所以已经过期的 key 都会从 keyspace 中删除。

具体就是 Redis 在每秒会做 10 次以下操作：
1. 测试随机 20 个 key 进行相关过期检测。
2. 删除所有已经过期的 key。
3. 如果有超过 25% 的 key 过期，重复步骤 1。

这是一个平凡的概率算法，基本上假设是，我们的样本代表了这个 key space，并且我们不断重复过期检测，直到过期的 key 的百分比低于 25%，这意味着，在任何给定时刻，最多会清除 1/4 的过期 key。

---

### How expires are handled in the replication link and AOF file

为了获得正确的行为而不牺牲一致性，当一个 key 过期，[DEL](del.md) 将会随着 AOF 文字一起合成到所有附加的 slaves。在 master 实例中，这种方法是集中的，并且不存在一致性错误的机会。

当然，当 slaves 连接到 master 时，不会独立过期 key（会等到 master 执行 [DEL](del.md) 命令），它们仍然会在数据集里面存着，所以当 slave 当选为 master 时淘汰 key 会独立执行，然后成为 master。