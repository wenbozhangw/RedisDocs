## Distributed locks with Redis

分布式锁(Distributed locks)在许多场景中是一个非常有用的原语，在这些环境中，不同的进程必须以互斥的方式使用共享资源。

有很多分布式锁的库和博客描述了如何使用 Redis 实现 DLM (分布式锁管理器，Distributed Lock Manager)，但每个库的实现方式都不太一样，很多库的实现方式为了简单降低了可靠性，而有些使用了稍微复杂的设计。

本页面尝试提供一种更规范的算法来使用 Redis 实现分布式锁。我们提出了一种称为 **Redlock** 的算法，它实现了我们认为比普通单实例方法更安全的 DLM。我们希望社区能帮助分析一下这种实现方法，并给我们提供反馈。

---

### Implementations

在我们开始描述算法之前，我们已经有一些可供参考的实现库。

- [Redlock-rb](https://github.com/antirez/redlock-rb) (Ruby implementation)。还有一个 [fork of Redlock-rb](https://github.com/leandromoreira/redlock-rb) ，它添加了一个 gem 以便于分发，可能还有更多。
- [Redlock-py](https://github.com/SPSCommerce/redlock-py) (Python implementation)
- [Pottery](https://github.com/brainix/pottery#redlock) (Python implementation)
- [Aioredlock](https://github.com/joanvila/aioredlock) (Asyncio Python implementation)
- [Redlock-php](https://github.com/ronnylt/redlock-php) (PHP implementation)
- [PHPRedisMutex](https://github.com/malkusch/lock#phpredismutex) (further PHP implementation)
- [cheprasov/php-redis-lock](https://github.com/cheprasov/php-redis-lock) (PHP library for locks)
- [](https://github.com/rtckit/reactphp-redlock) (Async PHP implementation).
- [Redsync](https://github.com/go-redsync/redsync) (Go implementation)
- [Redisson](https://github.com/mrniko/redisson) (Java implementation)
- [Redis::DistLock](https://github.com/sbertrang/redis-distlock) (Perl implementation)
- [Redlock-cpp](https://github.com/jacket-code/redlock-cpp) (C++ implementation)
- [Redlock-cs](https://github.com/kidfashion/redlock-cs) (C#/.NET implementation)
- [Redlock.net](https://github.com/samcook/RedLock.net) (C#/.NET implementation)。包括异步和锁扩展支持。
- [ScarletLock](https://github.com/psibernetic/scarletlock) (C#/.NET implementation with configurable datastore)
- [Redlock4Net](https://github.com/LiZhenNet/Redlock4Net) (C#/.NET implementation)
- [node-redlock](https://github.com/mike-marcacci/node-redlock) (NodeJS implementation)。包括锁扩展支持。

---

### Safety and Liveness guarantees

我们将仅使用三个属性对我们的设计进行建模，从我们的角度来看，这是以有效方式使用分布式锁所需要的最低保证。

1. 安全属性(Safety property)：独占互斥(Mutual exclusion)。在任何一个时刻，只有一个客户端持有锁。
2. 活性 A (Liveness property A)：无死锁 (Deadlock free)。即便持有锁的客户端崩溃（crashed）或者网络被分裂（gets partitioned），锁仍然可以被获取。
3. 活性 B (Liveness property B)：容错(Fault tolerance)。只要大部分 Redis 节点都或者，客户端就可以获取和释放锁。

---

### Why failover-based implementations are not enough

为了更好的理解我们想要改进的方面，我们先分析一下当前大多数基于 Redis 的分布式锁现状和实现方法。

实现 Redis 分布式锁的最简单方法就是在 Redis 中创建一个 key，这个 key 有一个失效时间(TTL)，以保证锁最终会被自动释放掉（这个对应特性 2）。当客户端释放资源（解锁）的时候，会删掉这个 key。

从表面上来看，似乎效果还不错，但是这里有一个问题：这个架构中存在一个严重的单点失败(single point of failure)问题。如果 Redis 挂了怎么办？你可能会说，可以通过增加一个 replica 节点解决这个问题。但这通常是行不通的。这样做，我们不能实现资源的独享，因为 Redis 的主从同步通常是异步的。

在这种场景（主从结构）中存在明显的竞争条件：

1. 客户端 A 从 master 获取到锁
2. 在 master 将锁同步到 replica 之前，master 宕机了。
3. replica 节点被晋升为 master 节点。
4. 客户端 B 取得了同一个资源被客户端 A 已经获取到的另一个锁。**SAFETY VIOLATION!**

有时候程序就是这么巧，比如说正好一个节点挂掉的时候，多个客户端同时取到了锁。如果你可以接受这种小概率错误，那用这个基于复制的方案就完全没有问题。否则的话，我们建议你实现下面描述的方案。

---

### Correct implementation with a single instance

在尝试克服上述单实例设置的限制之前，让我们先讨论一下这种简单情况下实现分布式锁的正确方法，实际上这是一种可行的方案，尽管存在竞态，结果仍然是可以接受的，另外，这里讨论的单实例加锁方法也是分布式加锁算法的基础。

获取锁使用命令：

```
SET resource_name my_random_value NX PX 30000
```

这个命令不仅在不存在 key 的时候才能被执行成功（NX 选项），并且这个 key 有一个 30 秒的自动失效时间（PX 选项）。这个 key 的值是 "my_random_value"（一个随机值），这个值在所有客户端必须是唯一的，所有同一 key 的获取者（竞争者）这个值都不能一样。

value 的值必须是随机数主要是为了更安全的释放锁，释放锁的时候使用脚本告诉 Redis：只有 key 存在并且存在的值和我指定的值一样才能告诉我删除成功。可以通过以下 Lua 脚本实现：

```lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

使用这种方式释放锁，可以避免删除别的客户端获取成功的锁。举个例子：客户端 A 取得资源锁，但是紧接着被一个其他操作阻塞了，当客户端 A 运行完毕其他操作后要释放锁时，原来的锁早已超时并且被 Redis 自动释放掉，并且在这期间，资源锁又被客户端 B 获取到。如果仅使用 DEL 命令将 key 删除，那么这种情况就会把客户端 B 的锁给删除掉。使用 Lua 脚本就不会存在这种情况，因为每个锁都用一个随机字符串“签名”，因此只有当它仍然是试图删除它的客户端设置的锁时，才会删除它。

这个随机字符串应该怎么设置？我认为它应该是从 /dev/urandom 产生的一个 20 字节随机数，但是我想你可以找到比这种方法代价更小的方法，只要这个数在你的任务中是唯一的就行。例如一种安全可行的方法是使用 /dev/urandom 作为 RC4 的种子和源产生一个伪随机流；一种更简单的方法是把以毫秒为单位的 unix 时间和客户端 ID 拼接起来，理论上不是完全安全，但是在多数情况下可以满足需求。

key 的失效时间，被称作 "lock validity time(锁定有效期)"。它不仅是 key 自动失效时间，而且还是一个客户端持有锁多次时间后可以被另一个客户端重新获得。

截止目前，我们已经有较好的方法获取锁和释放锁。基于 Redis 单实例，假设这个单实例总是可用的，这种方法已经足够安全。现在让我们扩展一下，假设 Redis 没有总是可用的保障。

---

### The Redlock algorithm

在 Redis 的分布式环境中，我们假设有 N 个 Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。之前我们已经描述了在 Redis 单实例下怎么安全地获取和释放锁。我们确保将在每 N 个实例上使用此方法获取和释放锁。在这个示例中，我们假设有 5 个 Redis master 节点，这是一个比较合理的假设，所以我们需要在 5 台机器上面或者 5 台虚拟机上面运行这些实例，这样保证它们不会同时都宕掉。

为了获取锁，客户端应该执行以下操作：

1. 获取当前 Unix 时间，以毫秒为单位。
2. 依次尝试从 N 个实例，使用相同的 key 和随机值获取锁。在步骤 2，当向 Redis 设置锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为 10 秒，则超时时间应该在 5-50 毫秒之间。这样可以避免服务器端 Redis 已经挂掉，或长期处于阻塞的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个 Redis 实例。
3. 客户端使用当前时间减去开始时间获取锁（步骤 1 记录的时间），就得到获取锁使用的时间。当且仅当从大多数（这里是 3 个节点）的 Redis 节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
4. 如果取到了锁，key 的真正有效时间等于有效时间减去获取锁使用的时间（步骤 3 计算的结果）。
5. 如果因为某些原因，获取锁失败（_没有_ 在至少 N/2+1 个 Redis 实例获取到锁或者获取锁时间已经超过了有效时间），客户端应该在所有的 Redis 实例上进行解锁（即便某些 Redis 实例根本就没有加锁成功）。

---

### Is the algorithm asynchronous ?

算法基于这样一个假设：虽然多个进程之间没有时钟同步，但每个进程都以相同的时钟频率前进，时间差相对于失效时间来说几乎可以忽略不计。这种假设和我们的真实世界非常接近：每个计算机都有一个本地时钟，我们可以容忍多个计算机之间有较小的时钟偏移。

从这点来说，我们必须再次强调我们的互相排斥规则：只有在锁的有效时间（在步骤 3 计算的结果）范围内，客户端能够做完它的工作，锁的安全性才能得到保证（锁的实际有效时间通常比设置的短，因为计算机之间有始终漂移的现象）。

想要了解更多关于需要 _clock drift(时钟漂移)_ 间隙的相似系统，这里有一个非常有趣的参考：

[Leases: an efficient fault-tolerant mechanism for distributed file cache consistency.](http://dl.acm.org/citation.cfm?id=74870)

---

### Retry on failure

当客户端无法获取到锁时，应该在一个 _随机_ 延迟后重试，防止多个客户端在 _同时_ 抢夺统一资源的锁（这样会导致脑裂，没有人会取到锁）。同样，客户端取得大部分 Redis 实例锁所花费的时间越短，脑裂出现的概率就会越低（必要的重试），所以，理想情况下，客户端应该同时（并发地）向所有 Redis 发送 SET 命令。

需要强调，当客户端从大多数 Redis 实例获取锁失败时，应该尽快地释放（部分）已经成功取到的锁，这样其他的客户端就不必非得等到锁过完 "有效时间" 才能取到（然而，如果已经存在网络分裂，客户端已经无法和 Redis 实例通信，此时就只能等待 key 的自动释放，等于被惩罚了）。

---

### Releasing the lock

释放锁比较简单，向所有的 Redis 实例发送释放锁命令即可，不用关心之前有没有从 Redis 实例成功获取到锁。

---

### Safety arguments

这是算法安全吗？我们可以从不同的场景讨论一下。

让我们假设客户端从大多数 Redis 实例获取到了锁。所有的实例都包含同样的 key，并且 key 的有效时间也一样。然而，key 肯定是在不同的时间被设置上的，所以 key 的失效时间也不是精确相同的。我们假设第一个设置的 key 的时间是 T1（开始向第一个服务端发送命令前时间），最后一个设置的 key 时间是 T2 (得到最后一台服务器答复后的时间)，我们可以确认，第一个服务器的 key 至少会存活 `MIN_VALIDITY = TTL - (T2 - T1) - CLOCK_DRIFT`。所有其他的 key 的存活时间，都会比这个 key 时间晚，所以可以肯定，所有 key 的失效时间至少是 MIN_VALIDITY 。

当大部分实例的 key 被设置后，其他的客户端将不能再获取到锁，因为至少 N/2 + 1 个实例已经存 key。所以，如果一个锁被客户端获取后，客户端自己也不能再次申请到锁（违反相互排斥属性）。

然而我们也想确保，当多个客户端同时抢夺一个锁时不能两个都成功。

如果客户端在获取到大多数 Redis 实例锁，使用的时间接近或者已经大于失效时间，客户端将认为锁是失效的锁，并且将释放掉已经获取到的锁，所以我们只需要在有效时间范围内获取到大部分锁这种情况。在上面已经讨论过有争议的地方，在 MIN_VALIDITY 时间内，将没有客户端再次取得锁。所以只有一种情况，多个客户端会在相同时间取得 N/2 + 1 实例的锁，那就是取得锁的时间大于失效时间（TTL），这样取到的锁也是无效的。

如果你能提供关于现有的类似算法的一个正式证明（指出正确性），或者发现这个算法的 bug？我们将非常感激。

---

### Liveness arguments

系统的活性安全基于三个主要特性：

1. 锁的自动释放（因为 key 失效了）：最终锁可以再次被使用。
2. 客户端通常会将没有获取到的锁删除，或者锁被获取到后，使用完之后会被客户端主动（提前）释放锁，而不是等到锁失效，另外的客户端才能取到锁。
3. 当客户端重试获取锁时，需要等待一段时间，这个时间必须大于从大多数 Redis 实例成功获取锁使用的时间，以最大限度的避免脑裂。

然而，当网络出现问题时，系统在失效时间（ [TTL](../commands/ttl.md)）内就无法服务，这种情况下我们的程序就会为此付出代价。如果忘了持续的有问题，可能就会出现死循环了。这种情况发生在当客户端取到一个锁还没有来得及释放锁就被网络隔离。

如果网络一直没有恢复，这个算法会导致系统不可用。

---

### Performance, crash-recovery and fsync

