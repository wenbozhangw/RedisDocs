## Transactions

相关命令:
- [DISCARD](../commands/DISCARD.md)
- [EXEC](../commands/EXEC.md)
- [MULTI](../commands/MULTI.md)
- [UNWATCH](../commands/UNWATCH.md)
- [WATCH](../commands/WATCH.md)

---

[MULTI](../commands/MULTI.md)、[EXEC](../commands/EXEC.md)、[DISCARD](../commands/DISCARD.md)、[WATCH](../commands/WATCH.md) 是 Redis 事务相关的命令。事务可以一次执行多个命令，并且带有一下两个重要保证：
- 事务是一个单独的隔离操作：事务中的所有命令都被序列化并按顺序执行。事务在执行过程**中**，不会被其他客户端发送来的命令请求所打断。这保证了命令作为单个隔离操作执行。
- 事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。[EXEC](../commands/EXEC.md) 命令会触发事务中的所有命令的执行，因此如果客户端在调用 [EXEC](../commands/EXEC.md) 命令之前在事务上下文中丢失了与服务器的连接，则不会执行任何操作，相反，如果调用 [EXEC](../commands/EXEC.md) 命令，执行所有操作。使用 [append-only](persistence.md#append-only-file) 文件时，Redis 确保使用单个 write(2) 系统调用将事务写入磁盘。但是，如果 Redis 服务器崩溃或被系统管理员以某种方式杀死，则可能只注册了部分操作。Redis 将在重新启动时检测到这种情况，并会出现错误退出。使用 `redis-check-aof` 工具可以修复的 append only 文件，将删除不完整的事务信息，以便服务器可以重新启动。

从 2.2 版本开始， Redis 还可以通过**乐观锁(optimistic lock)** 实现 **CAS(check-and-set)** 操作，具体信息请参考文档的后半部分。

---

### Usage

[MULTI](../commands/MULTI.md) 命令用于开启一个事务，他总是返回 OK。[MULTI](../commands/MULTI.md) 执行之后，客户端可以继续向服务器发送任意多条命令，这些命令不会立即被执行，而是被放到一个队列中，当 [EXEC](../commands/EXEC.md) 命令被调用时，所有队列中的命令才会被执行。

另一方面，通过调用 [DISCARD](../commands/DISCARD.md)，客户端可以清空事务队列，并放弃执行事务。

以下是一个事务例子，它原子地增加了 foo 和 bar 两个键的值：

```
> MULTI
OK
> INCR foo
QUEUED
> INCR bar
QUEUED
> EXEC
1) (integer) 1
2) (integer) 1
```

[EXEC](../commands/EXEC.md) 命令的回复是一个数组，数组中的每个元素都是执行事务中的命令所产生的的回复。其中，回复元素的先后顺序和命令发送的先后顺序一致。

当 Redis 连接处于 [MULTI](../commands/MULTI.md) 请求的上下文中时，所有传入的命令都会回复字符串 QUEUED（从 Redis 协议的角度来看，作为状态回复发送(status reply)）。排队的命令只是在调用 [EXEC](../commands/EXEC.md) 时执行。

---

### Errors inside a transaction

使用事务时可能会遇上以下两种错误：
- 事务在执行 [EXEC](../commands/EXEC.md) 之前，入队的命令可能会出错。比如说，命令可能会产生语法错误（参数数量错误，参数名错误，等等），或者其他更严重的错误，比如内存不足（如果服务器使用 `maxmemroy` 设置了最大内存限制的话）。
- 命令可能在 [EXEC](../commands/EXEC.md) 调用之后失败。举个例子，事务中的命令可能处理了错误类型的键，比如将列表命令用在了字符串键上，诸如此类。

对于发生在 [EXEC](../commands/EXEC.md) 执行之前的错误，客户端以前的做法是检查命令入队所得的返回值：如果命令入队时返回 QUEUED，那么入队成功；否则，就是入队失败。如果有命令在入队时失败，那么大部分客户端都会停止并取消这个事务。

不过，从 Redis 2.6.5 开始，服务器对命令入队失败的情况进行记录，并在客户端调用 [EXEC](../commands/EXEC.md) 命令时，拒绝执行并自动放弃这个事务。

在 Redis 2.6.5 之前，Redis 只执行事务中那些入队成功的命令，而忽略那些入队失败的命令。而新的处理方式则使得在 pipeline 中包含事务变得简单，因为发送事务和读取事务的回复都只需要和服务器进行一次通讯。

至于那些在 [EXEC](../commands/EXEC.md) 命令执行之后所产生的错误，并没有对它们进行特别处理：即使事务中有某个/某些命令在执行时产生了错误，事务中的其他命令仍然会继续执行。

从协议角度来看这个问题，会更容易理解一些。一下例子中， [LPOP](../commands/LPOP.md) 命令的执行将出错，尽管调用它的语法是正确的：

```
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
MULTI
+OK
SET a abc
+QUEUED
LPOP a
+QUEUED
EXEC
*2
+OK
-ERR Operation against a key holding the wrong kind of value
```

[EXEC](../commands/EXEC.md) 返回两条 [Bulk string reply](./protocol.md#resp-bulk-strings) ：第一条是 `OK`，而第二条是 `-ERR`。至于怎样使用合适的方法来表示事务中的错误，则是由客户端自己决定的。

最重要的是记住这一条，即使**事务中有某条/某些命令执行失败了，事务队列中的其他命令仍然会继续执行** —— Redis *不会*停止事务中的命令。

以下例子展示的是另一种情况， 当命令在入队时产生错误， 错误会立即被返回给客户端：

```
MULTI
+OK
INCR a b c
-ERR wrong number of arguments for 'incr' command
```

因为调用 [INCR](../commands/INCR.md) 命令的参数格式不正确，所以这个命令入队失败。

---

### Why Redis does not support roll backs?

如果你又使用关系型数据库的经验，那么 "Redis 在事务失败时不进行回滚，而是继续执行剩下的命令"这种做法可能会让你觉得有点奇怪。

以下是这种做法的优点：
- Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现），或是命令用在了错误类型的键上面：这也就是说，从实用性角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发过程中被发现，而不应该出现在生产环境中。
- 因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速。

有种观点认为 Redis 处理事务的做法会产生 bug，然而需要注意的是，在通常情况下，回滚并不能解决编程错误带来的问题。举个例子，如果你本来想通过 [INCR](../commands/INCR.md) 命令将键的值加上 1，却不小心加上了 2，又或者对错误类型的键执行了 [INCR](../commands/INCR.md) ，回滚是没有办法处理这些情况的。

---

### Discarding the command queue

当执行 [DISCARD](../commands/DISCARD.md) 命令时，事务会被放弃，事务队列会被清空，并且客户端会从事务状态中退出：

```
> SET foo 1
OK
> MULTI
OK
> INCR foo
QUEUED
> DISCARD
OK
> GET foo
"1"
```

---

### Optimistic locking using check-and-set

[WATCH](../commands/WATCH.md) 命令可以为 Redis 事务提供 check-and-set(CAS) 行为。

被 [WATCH](../commands/WATCH.md) 的键会被监视，并且会发觉这些键是否被改动过了。如果至少一个监视的键在 [EXEC](../commands/EXEC.md) 执行之前被修改了，那么整个事务都会被取消，[EXEC](../commands/EXEC.md) 返回 [Null reply](./protocol.md#null-bulk-string) 来表示事务已经失败。

举个例子，假设我们需要原子地为某个值进行加 1 操作（假设 [INCR](../commands/INCR.md) 不存在）。

首先我们可能会这样做：

```
val = GET mykey
val = val + 1
SET mykey $val
```

上面的这个实现在只有一个客户端的时候可以执行得很好。但是，当多个客户端同时对同一个键进行这样的操作时，就会差生竞争条件。举个例子，如果客户端 A 和 B 都读取了键原来的值，比如 10，那么两个客户端都会将键的值设置为 11，但正确的结果应该是 12 才对。

有了 [WATCH](../commands/WATCH.md)，我们就可以轻松地解决这类问题了：

```
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
```

使用上面的代码，如果在 [WATCH](../commands/WATCH.md) 执行之后，[EXEC](../commands/EXEC.md) 执行之前，有其他客户端修改了 `mykey` 的值，那么当前客户端的事务就会失败。

程序需要做的，就是不断重试这个操作，直到没有发生碰撞为止。这种形式的锁被称为 *乐观锁(optimistic locking)* ，它是一种非常请打的锁机制。并且在大多数情况下，不同的客户端会访问不同的键，碰撞的情况一般都很少， 所以通常并不需要进行重试。

---

### [WATCH](../commands/WATCH.md) explained

那么 [WATCH](../commands/WATCH.md) 到底是关于什么的呢？它会使 [EXEC](../commands/EXEC.md) 命令需要有条件的执行：我们要求 Redis 只有在没有修改任何被监视的 键时，才可以执行事务。这包括客户端所做的修改，如写入命令，以及 Redis 本身所做的修改，如过期和逐出。如果键再被监视和收到 [EXEC](../commands/EXEC.md) 之间被修改，整个事务将被中止。

**注意** 在 Redis 6.0.9 版本之前，过期的 key 不会导致事务中止。[More on this](https://github.com/redis/redis/pull/7920)。

事务中的命令不会触发 [WATCH](../commands/WATCH.md) 条件，因为它们只会在 [EXEC](../commands/EXEC.md) 发送之前 queued。

[WATCH](../commands/WATCH.md) 命令可以被调用多次。简单的说，所有的 [WATCH](../commands/WATCH.md) 调用都会监听从调用该命令开始到调用 [EXEC](../commands/EXEC.md) 位置，这期间的键的变化。您还可以向单个 [WATCH](../commands/WATCH.md) 调用传递任意数量的键。

当 [EXEC](../commands/EXEC.md) 被调用时，不管事务是否被中止，对所有的 key 的监听都会被取消。另外，当客户端连接关闭时，该客户端对键的监听也会被取消。

使用无参数的 [UNWATCH](../commands/UNWATCH.md) 命令客户手动取消对所有键的监听。对于一些需要改动多个键的事务，有时候程序需要同时对多个键进行加锁，然后检查这些键的当前值是否符合程序的要求。当值达不到要求时，就可以使用 [UNWATCH](../commands/UNWATCH.md) 命令来取消目前对键的监听，这样连接就可以自由地用于新事务。

---

#### Using [WATCH](../commands/WATCH.md) to implement ZPOP

[WATCH](../commands/WATCH.md) 可以用于创建 Redis 没有内置的原子操作。举个例子，Redis 不支持实现 ZPOP（ [ZPOPMIN](../commands/ZPOPMIN.md)、[ZPOPMAX](../commands/ZPOPMAX.md) 和它们的阻塞变体仅在 5.0 版本中添加 ），它可以原子地弹出有序集合中分值(score)最小的元素。这是最简单的实现：

```
WATCH zset
element = ZRANGE zset 0 0
MULTI
ZREM zset element
EXEC
```

如果 [EXEC](../commands/EXEC.md) 失败（即返回 [Null reply](./protocol.md#null-bulk-string)），我们只需重复该操作。

---

### Redis scripting and transactions

从定义上来说， [Redis script](../commands/EVAL.md) 本身就是一种事务，所以任何在事务里可以完成的事，在脚本里面也能完成。并且一般来说，使用脚本要来得更简单，并且速度更快。

因为脚本功能是 Redis 2.6 才引入的，而事务功能则更早之前就存在了，所以 Redis 才会同时存在两种处理事务的方法。不过我们并不打算在短时间内移除事务功能，因为事务提供了一种即使不使用脚本，也可以避免竞争条件的方法，而且事务本身的实现并不复杂。

不过在不远的将来，可能所有用户都会只使用脚本来实现事务也说不定。如果真的发生这种情况的话，那么我们将废弃并最终移除事务功能。