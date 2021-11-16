## EVAL script numkeys key [key [key ...]] [arg [arg ...]]

    起始版本：2.6.0
    时间复杂度：取决于脚本本身的执行的时间复杂度。

---

### Introduction to EVAL

[EVAL](eval.md) 和 [EVALSHA](evalsha.md) 命令是从 Redis 2.6.0 版本开始，使用内置的 Lua 解释器，可以对 Lua 脚本进行求值。

[EVAL](eval.md) 的第一个参数是一段 Lua 5.1 脚本程序。这段 Lua 脚本不需要（也不应该）定义函数。它运行在 Redis 服务器中。

[EVAL](eval.md) 的第二个参数是参数的个数，后面的参数（从第三个参数），表示在脚本中所用到的那些 Redis key，这些 key 参数可以在 Lua 中通过全局变量 KEYS 数组，用 1 为初始值的形式方位（KEYS[1]、KEY[2]，以此类推）。

在命令的最后，那些不是 key 参数的附加参数 arg [arg ...]，可以在 Lua 中通过全局变量 ARGV 数组访问，访问的形式和 KEYS 变量类似（ARGV[1]、ARGV[2]，诸如此类）。

举例说明：

```
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

注：返回结果是 Redis multi bulk replies 的 Lua 数组，这是一个 Redis 的返回类型，您的客户端可能会将它们转换成数组类型。

这是从一个 Lua 脚本中使用两个不同的 Lua 函数来调用 Redis 命令的例子：

- redis.call()
- redis.pcall()

redis.call() 与 redis.pcall() 很类似，它们唯一的区别是当 redis 命令执行结果返回错误的时候，redis.call() 将返回给调用者一个错误，而 redis.pcall() 会将捕获的错误以 Lua table的形式返回。

redis.call() 和 redis.pcall() 两个函数的参数可以是任意的 Redis 命令：

```
> eval "return redis.call('set','foo','bar')" 0
OK
```

需要注意的是，上面这段脚本的确实现了将 key foo 的值设置为 bar 的目的。但是，他违反了 [EVAL](eval.md) 命令的语义，因为脚本里使用的所有 key 都应该由 KEYS 数组来传递，就像这样：

```
> eval "return redis.call('set',KEYS[1],'bar')" 1 foo
OK
```

必须使用正确的形式来传递键，才能确保分析工作正确地执行。因此，为了使 [EVAL](eval.md) 正确执行，必须显式传递 key。这在很多方面都很有用，尤其是确保 Redis Cluster 可以将您的请求转发到适当的集群节点。

不过，这条规矩并不是强制性的， 从而使得用户有机会滥用(abuse) Redis 单实例配置(single instance configuration)， 代价是这样写出的脚本不能被 Redis Cluster 兼容。

Lua 脚本能返回一个值，这个值能按照一组转换规则从Lua转换成redis的返回类型。

---

### Conversion between Lua and Redis data types

当 Lua 通过 call() 或 pcall() 函数执行 Redis 命令的时候，命令的返回值会被转换成 Lua 数据结构。同样地，当 Lua 脚本在 Redis 内置的解释器里运行时，Lua 脚本的返回值也会被转换成 Redis 协议，然后由 [EVAL](eval.md) 将值返回给客户端。

数据类型之间的转换遵循这样的一个设计原则：如果将一个 Redis 值转换成 Lua 值，之后再将转换所得的 Lua 值转换回 Redis 值，那么这个转换所得的 Redis 值应该和最初时的 Redis 值一样。

换句话说，Lua 类型和 Redis 类型之间存在着一一对应的转换关系。

**Redis 到 Lua**的转换表。

- Redis integer reply -> Lua number
- Redis bulk reply -> Lua string
- Redis multi bulk reply -> Lua table (may have other redis data types nested)（Redis 多条 bulk 回复转换成 Lua 表，表内可能有其他别的 Redis 数据类型）
- Redis status reply -> Lua table with a single ok field containing the status （Redis 状态回复转换成 Lua 表，表内的 ok 域包含了状态信息）
- Redis error reply -> Lua table with a single err field containing the error （Redis 错误回复转换成 Lua 表，表内的 err 域包含了错误信息）
- Redis Nil bulk reply and Nil multi bulk reply -> Lua false boolean type （Redis 的 Nil 回复和 Nil 多条回复转换成 Lua 的布尔值 false）

**Lua 到 Redis**的转换表。

- Lua number -> Redis integer reply (the number is converted into an integer)（ Lua 数字转换成 Redis 整数）
  Lua string -> Redis bulk reply （Lua 字符串转换成 Redis bulk 回复）
- Lua table (array) -> Redis multi bulk reply (truncated to the first nil inside the Lua array if any)（Lua 表(数组)转换成 Redis 多条 bulk 回复）
- Lua table with a single ok field -> Redis status reply （一个带单个 ok 域的 Lua 表，转换成 Redis 状态回复）
- Lua table with a single err field -> Redis error reply （一个带单个 err 域的 Lua 表，转换成 Redis 错误回复）
- Lua boolean false -> Redis Nil bulk reply.（Lua 的布尔值 false 转换成 Redis 的 Nil bulk 回复）

从 Lua 转换到 Redis 有一条额外的规则，这条规则没有和它对应的从 Redis 转换到 Lua 的规则：

- Lua boolean true -> Redis integer reply with value of 1.（Lua 布尔值 true 转换成 Redis 整数回复中的 1）

还有下面两点需要重点注意：

- Lua 中整数和浮点数之间没有什么区别。因此，无门始终将 Lua 的数字转换成整数回复，这样将舍去小数部分。**如果你想从 Lua 返回一个浮点数，你应该将它作为一个字符串**（比如[ZSCORE](zscore.md) 命令）。
- There is [no simple way to have nils inside Lua arrays](http://www.lua.org/pil/19.1.html) ，这是 Lua table 语义的结果，因此当 Redis 将 Lua 数组转换为 Redis 协议时，如果遇到 nil，则转换会停止。
- 当 Lua table 包含 key（以及值）时，转换后的 Redis 回复将**不**包含它们。

**RESP3 mode conversion rules:** 请注意，Lua 引擎可以使用新的 Redis 6 协议在 RESP3 模式下工作。在这种情况下，有额外的转换规则，并且与 RESP2 模式相比，某些转换也进行了修改。有关更多信息，请参阅本文档的 RESP3 部分。

以下是几个类型转换的例子：

```
> eval "return 10" 0
(integer) 10

> eval "return {1,2,{3,'Hello World!'}}" 0
1) (integer) 1
2) (integer) 2
3) 1) (integer) 3
   2) "Hello World!"

> eval "return redis.call('get','foo')" 0
"bar"
```

最后一个例子展示了如何从 Lua 接收 redis.call() 或 redis.pcall() 的准确返回值，如果直接调用该命令将返回该值。

下面的例子我们可以看到浮点数和nil将怎么样处理：

```
> eval "return {1,2,3.3333,'foo',nil,'bar'}" 0
1) (integer) 1
2) (integer) 2
3) (integer) 3
4) "foo"
```

如你看到的 3.333 被转换成了3，_somekey_ 被排除掉，并且 nil后面的字符串bar没有被返回回来。

---

### Helper functions to return Redis types

有两个辅助函数从 Lua 返回 Redis 的类型。

- `redis.error_reply(error_string)` 返回一个 error reply。此函数仅返回 single field table，其中 err 字段是您设置的指定的字符串。
- `redis.status_reply(status_string)` 返回 status reply。此函数仅返回 single field table，其中 ok 字段是您设置的指定的字符串。

使用辅助函数和直接返回指定格式的 table 没有区别，所以下面两种形式是等价的：

```
return {err="My Error"}
return redis.error_reply("My Error")
```

---

### Atomicity of scripts

Redis 使用单个 Lua 解释器去运行所有脚本，并且，Redis 也保证脚本会以原子性(atomic)的方式执行：当某个脚本在运行的时候，不会有其他脚本或 Redis 命令被执行。这和使用 [MULTI](multi.md)/[EXEC](exec.md) 块执行的事务很类似。在其他别的客户端看来，脚本的效果(effect)要么是不可见的(not visible)，要么就是已完成的(already completed)。

另一方面，这也意味着，执行一个运行缓慢的脚本并不是一个好主意。创建快速脚本并不难，因为脚本开销非常低，但是如果您打算使用慢速脚本，您应该意识到在脚本运行时没有其他客户端可以执行命令。

---

### Error handling

如前所述，`redis.call()`在执行命令过程中发送错误时，脚本会停止执行，并返回一个脚本错误，错误的输出信息会说明错误造成的原因：

```
> del foo
(integer) 1
> lpush foo a
(integer) 1
> eval "return redis.call('get','foo')" 0
(error) ERR Error running script (call to f_6b1bf486c81ceb7edf3c093f4c48582e38c0e791): ERR Operation against a key holding the wrong kind of value
```

使用 `redis.pcall()` 不会引发错误，但会以上面指定的格式返回错误对象（返回一个带 err 域的 Lua 表(table)）。该脚本可以通过 `redis.pacll()` 返回的错误对象，将真正的错误返回给用户。

---

### Running Lua under low memory conditions

当 Redis 中的内存使用量超过 `maxmemory` 限制时，Lua脚本中遇到的第一个使用额外内存的写命令将导致脚本中止（除非使用 redis.pcall）。但是，这里需要注意一点是，如果第一个写入命令没有使用额外的内存，例如 DEL、LREM 或 SREM 等，Redis 将允许它运行，并且为了保证原子性， Lua 脚本中的所有后续命令都将执行完成。如果脚本中的后续写入生成额外的内存，Redis 内存使用量可能会超过 `maxmemory`。Lua 脚本导致 Redis 内存使用量超过 `maxmemory` 的另一种可能方式发生在脚本执行开始时，Redis 略低于 `maxmemory`，因此允许脚本中的第一个写入命令。随着脚本的执行，后续的写入命令会使用内存并导致 Redis 服务超过 `maxmemory`。

在这些情况下，建议将 `maxmemory-policy` 配置为不使用 `noeviction`。Lua脚本也应该很短，以便在 Lua 脚本之间可以发生逐出。

---

### Bandwidth and EVALSHA

[EVAL](eval.md) 命令要求你在每次执行脚本的时候都发送一次脚本主体(script body)。Redis 有一个内部的缓存机制，因此它不会每次都重新编译脚本，不过在很多场合，使用不必要的带宽来传送脚本主体并不是最佳选择。

另一方面，由于以下几个原因，使用特殊命令或通过 redis.conf 定义命令将是一个问题：

- 不同的实例可能有不同的命令实现。
- 如果我们必须确保所有实例都包含一个给定的命令，那么部署是很困难的，尤其是在分布式环境中。
- 阅读应用程序代码，完整的语义可能不清楚，因为应用程序调用服务器端定义的命令。

为了减少带宽损失的同时，避免上述问题，Redis 实现了 [EVALSHA](evalsha.md) 命令。

[EVALSHA](evalsha.md) 的工作方式与 [EVAL](eval.md) 完全一样，但是它接受的第一个参数不是脚本，而是脚本的 SHA1 摘要(SHA1 digest)。行为如下：

- 如果服务器还记得给定的 SHA1 摘要的脚本，则执行该脚本。
- 如果服务器不记得带有此 SHA1 摘要的脚本，则会返回一个特殊错误，告诉客户端改用 [EVAL](eval.md) 。

示例：

```
> set foo bar
OK
> eval "return redis.call('get','foo')" 0
"bar"
> evalsha 6b1bf486c81ceb7edf3c093f4c48582e38c0e791 0
"bar"
> evalsha ffffffffffffffffffffffffffffffffffffffff 0
(error) NOSCRIPT No matching script. Please use EVAL.
```

客户端库实现了可以一直乐观地使用 [EVALSHA](evalsha.md) 来代替 [EVAL](eval.md)，并期望要使用的脚本已经保存在服务器上了，只有当 **NOSCRIPT** 错误发生时，才使用 [EVAL](eval.md) 命令重新发送脚本，这样就可以最大限度地节省带宽。

传递 key 和参数作为额外的 [EVAL](eval.md) 参数在这种情况下也非常有用，因为脚本字符串保持不变，可以被 Redis 有效的缓存。

---

### Script cache semantics

Redis 保证所有运行过的脚本都会被永久保存在脚本缓存中，这意味着，当 [EVAL](eval.md) 命令在一个 Redis 实例上运行成功之后，随后针对这个脚本的所有 [EVALSHA](evalsha.md) 命令执行成功。

脚本可以被长时间缓存的原因是，编写好的应用程序不太可能有非常多的不同脚本来导致内存问题。每个脚本在概念上就像一个新命令的实现，甚至一个大型应用程序也可能只有几百个。即使应用程序被多次修改，脚本会发生变化，使用的内存也可以忽略不计。

刷新脚本缓存的唯一方法是显式地调用 [SCRIPT FLUSH](script-flush.md) 命令，该命令将完全刷新脚本缓存，删除目前执行的所有脚本。

通常只有在云计算环境中，Redis 实例被改作其他客户或者别的应用程序的实例时，才会执行这个命令。

此外，如上所述，重新启动 Redis 实例会刷新脚本缓存，因为缓存不是持久的。然而，从客户端的角度来看，只有两种方法可以确保 Redis 实例不会在两个不同命令之间重新切换。

- 我们与服务器的连接是持久的，到目前为止从未关闭。
- 客户端显式检查 [INFO](info.md) 命令中的 `runid` 字段，以确保服务器没有重新启动并且仍然是相同的进程。

实际上，对于客户端来说，最好简单地假设在给定连接的上下文中，除非管理员明确调用 [SCRIPT FLUSH](script-flush.md) 命令，否则保证缓存脚本存在。

用户可以假设 Redis 不会删除脚本，这在 pipeline 环境中的语义上很有用。

例如，对于一个和 Redis 保持持久化连接（persistent connection）的程序来说，他可以确信，执行过一次的脚本会一直保留在内存当中，因此他可以在 pipeline 中使用 EVALSHA 命令而不必担心因为找不到所需的脚本而产生错误（我们稍后会详细介绍这个问题）。

一种常见的模式是调用 [SCRIPT LOAD](script-load.md) 来加载将会在 pipeline 中使用到的所有脚本，然后直接在 pipeline 内使用 [EVALSHA](evalsha.md)，而无需检查因无法识别脚本哈希而导致的错误。

---

### The SCRIPT command

Redis 提供了一个 *SCRIPT* 命令，可用于控制脚本子系统(scripting subsystem)。SCRIPT 目前接受三种不同的命令：

- [SCRIPT FLUSH](script-flush.md) \
  此命令是强制 Redis 刷新脚本缓存的唯一方法。它在将同一实例重新分配给不同用户的云环境中最为有用。它也可用于测试客户端库的脚本功能实现。
- SCRIPT EXISTS sha1 sha2 ... shaN
  给定一个 SHA1 摘要列表作为参数，此命令返回一个 1 或 0 的数组，其中 1 表示指定的 SHA1 是否在脚本缓存中存在，而 0 表示没有在缓存中的 SHA1 脚本（或者至少在最新的 SCRIPT FLUSH 命令之后从未见过）。
- SCRIPT LOAD script
  该命令在 Redis 脚本缓存中注册指定的脚本。此命令在我们想要确保 [EVALSHA](evalsha.md) 不会因为脚本不存在执行失败时很有用（例如在 pipeline 和 MULTI/EXEC 操作期间），而无需实际执行脚本。
- [SCRIPT KILL](script-kill.md)
  如果脚本执行时间达到为此脚本配置的最大执行时间，此命令是中断长时间运行的脚本的唯一方法。SCRIPT KILL 命令只能用于在执行期间未修改数据集(dataset)的脚本（因为停止只读脚本不会违反脚本引擎保证的原子性）。有关长时间运行的脚本的更多信息，请参阅下一节。

---

### Scripts with deterministic writes

注意：从 Redis 5 开始，脚本在集群复制时，总是复制执行结果(scripts are always replicated as effects)，而不是逐字发送脚本。因此，以下部分主要适用于 Redis 4 或更早版本。

脚本的一个非常重要的部分是编写只以确定的方式改变数据库的脚本。默认情况下，在 Redis 实例中执行的脚本通过发送脚本本身（而不是结果命令）传播到副本和 AOF 文件。由于脚本将在远程主机上重新运行（或在重新加载 AOF 文件时），因此他对数据库所做的更改必须是可重现的。

发送脚本的原因是它通常比发送脚本生成的多个命令要快得多。如果客户端向主服务器发送许多脚本，将脚本转换为 replica/AOF 的单个命令，将会因为 replication link 或 Append Only File 导致过多的带宽占用（并且由于调度通过网络接收到的命令，因此 CPU 也过多，与调度由 Lua 脚本调用的命令相比，Redis 需要做更多的工作）。

通常情况下，复制脚本比复制脚本的执行结果更有意义，但并非在所有情况下都如此。因此，从 Redis 3.2 开始，脚本引擎能够复制脚本执行产生的写入命令序列，而不是复制脚本本身。有关更多信息，请参阅下一节。

在本节中，我们结社通过发送整个脚本在集群中同步脚本 replicated。我们称这种复制模式为 **whole scripts replication**。

whole scripts replication 方法的主要缺点是脚本需要具有以下属性：
- 给定相同地输入数据集，脚本必须始终使用相同的参数执行相同的 Redis 写入命令。脚本执行的操作不能依赖于任何隐藏（非显式）信息或状态，这些信息或状态可能随着脚本执行的进行或在脚本的不同执行之间而改变，也不能依赖于来自 I/O 设备的任何外部输入。

使用系统时间、调用 Redis 随机命令（如 [RANDOMKEY](randomkey.md)）或使用 Lua 的随机数生成器之类的事情可能导致脚本不会总是以相同的方式执行。

为了在脚本中强制执行此行为，Redis 执行以下操作：
- Lua 不导出访问系统时间或者其他外部状态的命令。
- 如果脚本会在 Redis 随机命令之后（如 [RANDOMKEY](randomkey.md)、[SRANDMEMBER](srandmember.md)、[TIME](time.md)）调用 Redis 指令，Redis 将阻止脚本并显示错误。这意味着如果脚本是只读的并且不修改数据集，则可以自由调用这些命令。请注意，随机命令不一定意味着使用随机数的命令：任何不确定的命令都被认为是随机命令(在这方面最好的例子是 [TIME](time.md) 命令)。
- 在 Redis 4 版本中，可能会以随机顺序返回元素的命令，例如 [SMEMBERS](smembers.md) （因为 Redis 集合是无序的）在从 Lua 调用时具有不同的行为，并且在将数据返回到 Lua 脚本之前会经历一个无声的自动减排序过滤器(silent lexicographical sorting filter)。所以 `redis.call("smembers", KEYS[1])` 将始终以相同的顺序返回 Set 元素，但是，从普通客户端调用的命令可能会返回不同的结果，即使 key 包含完全相同的元素。然而，从 Redis 5 开始，不再有这样的排序步骤，因为 Redis 5 以一种不再需要将非确定性命令转换为确定性命令的方式复制脚本。一般来说，即使在为 Redis 4 开发时，也不要假设 Lua 中的某些命令会被排序，而是依赖于你调用的原始命令的文档来查看它提供的属性。
- Lua 的伪随机数生成函数 `math.random` 被修改为每次执行新脚本时总是使用相同的种子。这意味着如果不使用 `math.randomseed`，则每次执行脚本时调用 `math.random` 将始终生成相同的数字序列。

然而，用户仍然可以使用以下简单的技巧编写具有随机行为的命令。想象一下，我想编写一个 Redis 脚本，用 N 个随机整数填充一个列表。

我可以从这个小 Ruby 程序开始：

```ruby
require 'rubygems'
require 'redis'

r = Redis.new

RandomPushScript = <<EOF
    local i = tonumber(ARGV[1])
    local res
    while (i > 0) do
        res = redis.call('lpush',KEYS[1],math.random())
        i = i-1
    end
    return res
EOF

r.del(:mylist)
puts r.eval(RandomPushScript,[:mylist],[10,rand(2**32)])
```

这个程序每次运行都会生成带有以下元素的列表：

```
> lrange mylist 0 -1
 1) "0.74509509873814"
 2) "0.87390407681181"
 3) "0.36876626981831"
 4) "0.6921941534114"
 5) "0.7857992587545"
 6) "0.57730350670279"
 7) "0.87046522734243"
 8) "0.09637165539729"
 9) "0.74990198051087"
10) "0.17082803611217"
```

为了使其具有确定性的同时，仍要确保脚本的每次调用都会产生不同的随机元素，我们可以简单地向脚本添加一个额外的参数，该参数将用于为 Lua 伪随机数生成器提供种子。以下是修改后的脚本：

```ruby
RandomPushScript = <<EOF
    local i = tonumber(ARGV[1])
    local res
    math.randomseed(tonumber(ARGV[2]))
    while (i > 0) do
        res = redis.call('lpush',KEYS[1],math.random())
        i = i-1
    end
    return res
EOF

r.del(:mylist)
puts r.eval(RandomPushScript,1,:mylist,10,rand(2**32))
```

我们在这里所做的是发送种子 PRNG 作为参数之一。给定相同的参数（我们的要求），脚本输出将始终相同，但我们在每次调用时更改其中一个参数，生成 random seed client-side。种子将作为 replication link 和 Append Only File 中的参数之一传播，确保在重新加载 AOF 或 replica 处理脚本时将生成相同地更改。

注意：此行为的一个重要部分是，无论运行 Redis 的系统架构如何，Redis 实现为 `match.random` 和 `math.randomseed` 的 PRNG 都保证具有相同地输出。32 位、64 位、大端和小端系统都将产生相同地输出。

---

### Replicating commands instead of scripts

注意：从 Redis 5 开始，本节描述的复制方式(scripts effects replication)是默认的，不需要显式启用。

从 Redis 3.2 开始，可以选择替代复制(alternative replication)方法。我们可以支付至脚本生成的单个写入命令，而不是复制整个脚本。我们称之为 **script effects replication**。

在这种复制模式下，在执行 Lua 脚本的同时，Redis 会收集 Lua 脚本引擎执行的所有实际修改数据集的命令。当脚本执行完成时，脚本生成的命令序列被包装到 MULTI/EXEC 事务中并发送到副本和 AOF。

根据用例，这在多种方面都很有用：
- 当脚本计算速度慢，但可以通过几个写命令总结效果时，在副本上或重新加载 AOF 时重新计算脚本是一种 shame。在这种情况下，最好只复制脚本的效果。
- 启用脚本效果复制(script effects replication)后，将取消对非确定函数的限制。例如，您可以在脚本中的任何地方自由使用 [TIME](time.md) 或 [SRANDMEMBER](srandmember.md) 命令。
- 这种模式下的 Lua PRNG 在每次调用时种子是随机的。

要启用脚本效果复制，您需要在脚本执行写入之前发出以下 Lua 命令：

```lua
redis.replicate_commands()
```

如果启用了脚本效果复制，则该函数返回 true；否则，如果在脚本已经调用了写入命令之后调用该函数，则返回 false，并使用正常的整个脚本复制(whole script replication)。

---

### Selective replication of commands

选择脚本效果复制（参见上一节），可以更好的以命令方式传播到副本和AOF。这是一个非常高级的功能，因为误用可能会破坏 master、replicas 和 AOF 必须都包含相同逻辑内容的约定，从而造成损害。

然而，这是一个有用的功能，因为有时我们只需要在主服务器中执行某些命令才能创建，例如，中间值。

想想我们在两个集合之间执行交集的 Lua 脚本。然后我们从交集中挑选五个随机元素并创建一个包含它们的新集合。最后，我们删除代表两个原始集合之间交集的临时 key。我们想要复制的只是具有五个元素的新集合的创建。复制创建临时 key 的命令也没有用。

为此，Redis 3.2 引入了一个新命令，该命令仅在启用脚本效果复制时有效，并且能够控制脚本复制引擎。该命令称为 `redis.set_repl()` 并且如果在禁用脚本效果复制时调用，则会失败并引发错误。

可以使用四个不同的参数调用该命令：

```
redis.set_repl(redis.REPL_ALL) -- Replicate to the AOF and replicas.
redis.set_repl(redis.REPL_AOF) -- Replicate only to the AOF.
redis.set_repl(redis.REPL_REPLICA) -- Replicate only to replicas (Redis >= 5)
redis.set_repl(redis.REPL_SLAVE) -- Used for backward compatibility, the same as REPL_REPLICA.
redis.set_repl(redis.REPL_NONE) -- Don't replicate at all.
```

默认情况下，脚本引擎设置为 `REPL_ALL`。通过调用此函数，用户可以随时打开或关闭复制模式。

一个简单的例子如下：

```
redis.replicate_commands() -- Enable effects replication.
redis.call('set','A','1')
redis.set_repl(redis.REPL_NONE)
redis.call('set','B','2')
redis.set_repl(redis.REPL_ALL)
redis.call('set','C','3')
```

运行上述脚本后，结果是在副本和AOF上只会创建键A和C。

---

### Global variables protection

为了防止不必要的数据泄露进 Lua 环境，Redis 脚本不允许创建全局变量。如果一个脚本需要在多次执行之间维持某种状态（一种非常不常见的需求），他应该使用 Redis key 来进行状态保存。

当尝试访问全局变量时，脚本终止并且 EVAL 返回错误：

```
redis 127.0.0.1:6379> eval 'a=10' 0
(error) ERR Error running script (call to f_933044db579a2f8fd45d8065f04a8d0249383e57): user_script:1: Script attempted to create global variable 'a'
```

访问不存在的全局变量会产生类似的错误。

Lua 的 debug 工具，或者其他设施，比如打印（alter）用于实现全局保护的 meta table ，都可以用于实现全局变量保护。

实现全局变量保护并不难，不过有时候还是会不小心而为之。一旦用户在脚本中混入了 Lua 全局状态，那么 AOF 持久化和复制（replication）都会无法保证，所以，请不要使用全局变量。

Lua 新手请注意：为了避免在脚本中使用全局变量，只需使用 `local` 关键字声明要使用的每个变量。

---

### Using SELECT inside scripts

在正常的客户端连接里面可以调用 [SELECT](select.md) 选择内部的 Lua 脚本，但是 Redis 2.8.11 和 Redis 2.8.12 在行为上有一个微妙的变化。在 2.8.12 之前，会将脚本传送到调用脚本的当前数据库。从 2.8.12 开始，Lua脚本只影响脚本本身的执行，但不修改当前客户端调用脚本时选定的数据库。

从补丁级发布的语义变化是必要的，因为旧的行为与Redis复制层固有的不相容是错误的原因。

---

### Using Lua scripting in RESP3 mode

从 Redis 6 版本开始，服务器支持两种不同的协议。

一种成为 RESP2，是旧协议：到服务器的所有新连接都以这种模式启动。然而，客户端可以使用 [HELLO](hello.md) 命令写上新协议：这样连接就可以使用 RESP3 模式。在这种模式下，某些命令（例如 [HGETALL](hgetall.md)）使用新数据类型（在此特定情况下为 Map 数据类型）回复。RESP3 协议在语义上更强大，但是大多数脚本只是用 RESP2 就可以了。

Lua 引擎在于 Redis 通信时总是假定以 RESP2 模式运行，因此无论调用 [EVAL](eval.md) 或 [EVALSHA](evalsha.md) 命令的连接处于 RESP2 或 RESP3 模式，当使用 `redis.call()` 内置函数调用命令时，从 Redis 中查看，Lua 脚本默认情况下仍会看到它们使用相同类型的回复。

然而，在 Redis 6 或更高版本中运行的 Lua 脚本能够切换到 RESP3 模式，并使用新地可用类型获取回复。类似的，Lua脚本能够使用新类型回复客户端。在继续阅读本节之前，请确保了解 [the capabilities for RESP3](https://github.com/antirez/resp3) 。

为了切换到 RESP3，脚本应该调用这个函数：

```
redis.setresp(3)
```

请注意，通过使用参数“3”或“2”调用函数，脚本可以在 RESP3 和 RESP2 之间来回切换。

此时，新地转换可用，特别是：

**Redis to Lua**，RESP3特有的转换表：
- Redis map reply -> Lua table 具有单个映射字段，其中包含一个表示映射字段的值的 Lua table。
- Redis set reply -> Lua table 具有单个集合字段，其中包含一个 Lua table，该表将集合的元素表示为字段，其值为 true。
- Redis new RESP3 single null value -> Lua nil。
- Redis true reply -> Lua true boolean value.
- Redis false reply -> Lua false boolean value.
- Redis double reply -> Lua table，包含一个表示双精度值的 Lua 数字的 single score 字段。
- Redis big number reply -> Lua table 中有一个 `big_number` 字段，其中包含一个标识大数值的 Lua 字符串。
- Redis verbatim string reply -> Lua table 具有单个 `verbatim_string` 字段，其中包含一个 Lua table，该表具有两个字段，字符串和格式，分布表示 verbatim string 和 verbatim format respectively。
- 所有 RESP2 旧转换仍然适用。

注意：大数字和逐字回复仅在 Redis 7 或更高版本中可用。此外，目前 Lua 不支持 RESP3 属性。

**Lua to Redis**，特定的 RESP3 转换表：
- Lua boolean -> Redis boolean true or false. **注意，与 RESP2 模式相比，这是一个变化**，其中从 Lua 返回 true 时将数字 1 返回给 Redis 客户端，而返回 false 使用 NULL。
- Lua table，其中一个 map 字段被设置为 field-value Lua table -> Redis map reply.
- Lua table，其中一个 set 字段被设置为 field-value Lua table -> Redis set reply, 值被丢弃并且可以是任何值。
- Lua table，其中一个 double 字段被设置为 field-value Lua table -> Redis double reply.
- Lua null -> Redis RESP3 new null reply (protocol "_\r\n").
- 除非以上指定，否则所有 RESP2 旧转换仍然适用。

有一点需要理解：如果 Lua 回复 RESP3 类型，但调用 Lua 的连接处于 RESP2 模式， Redis 会自动将 RESP3 协议转换为 RESP2 兼容协议，就像正常命令发生那样。例如，在 RESP2 模式下将 map 类型返回到连接将具有返回字段和值的平面数组的效果。

---

### Available libraries

Redis Lua 解释器加载以下 Lua 库：
- `base` lib.
- `table` lib. 
- `string` lib. 
- `math` lib. 
- `struct` lib. 
- `cjson` lib. 
- `cmsgpack` lib. 
- `bitop` lib. 
- `redis.sha1hex` function. 
- `redis.breakpoint` and `redis.debug` function in the context of the [Redis Lua debugger](../topics/ldb.md).

每一个Redis实例都拥有以上的所有类库，以确保您使用脚本的环境都是一样的。

struct, CJSON 和 cmsgpack 都是外部库, 所有其他库都是标准 Lua 库。

#### struct

struct 是一个Lua装箱/拆箱 (packing/unpacking) 的库。

```
Valid formats:
> - big endian
< - little endian
![num] - alignment
x - pading
b/B - signed/unsigned byte
h/H - signed/unsigned short
l/L - signed/unsigned long
T   - size_t
i/In - signed/unsigned integer with size `n' (default is size of int)
cn - sequence of `n' chars (from/to a string); when packing, n==0 means
     the whole string; when unpacking, n==0 means use the previous
     read number as the string length
s - zero-terminated string
f - float
d - double
' ' - ignored
```

例如：

```
127.0.0.1:6379> eval 'return struct.pack("HH", 1, 2)' 0
"\x01\x00\x02\x00"
127.0.0.1:6379> eval 'return {struct.unpack("HH", ARGV[1])}' 0 "\x01\x00\x02\x00"
1) (integer) 1
2) (integer) 2
3) (integer) 5
127.0.0.1:6379> eval 'return struct.size("HH")' 0
(integer) 4
```

#### CJSON

CJSON 库为Lua提供极快的JSON处理。

例如：

```
redis 127.0.0.1:6379> eval 'return cjson.encode({["foo"]= "bar"})' 0
"{\"foo\":\"bar\"}"
redis 127.0.0.1:6379> eval 'return cjson.decode(ARGV[1])["foo"]' 0 "{\"foo\":\"bar\"}"
"bar"
```

#### cmsgpack

cmsgpack 库为Lua提供了简单、快速的MessagePack操纵。

例如：

```
127.0.0.1:6379> eval 'return cmsgpack.pack({"foo", "bar", "baz"})' 0
"\x93\xa3foo\xa3bar\xa3baz"
127.0.0.1:6379> eval 'return cmsgpack.unpack(ARGV[1])' 0 "\x93\xa3foo\xa3bar\xa3baz"
1) "foo"
2) "bar"
3) "baz"
```

#### bitop

bitop库为Lua的位运算模块增加了按位操作数。 它是Redis 2.8.18开始加入的。

例如：

```
127.0.0.1:6379> eval 'return bit.tobit(1)' 0
(integer) 1
127.0.0.1:6379> eval 'return bit.bor(1,2,4,8,16,32,64,128)' 0
(integer) 255
127.0.0.1:6379> eval 'return bit.tohex(422342)' 0
"000671c6"
```

它支持几个其他功能： bit.tobit, bit.tohex, bit.bnot, bit.band, bit.bor, bit.bxor, bit.lshift, bit.rshift, bit.arshift, bit.rol, bit.ror, bit.bswap. 所有可用的功能请参考 [Lua BitOp documentation](http://bitop.luajit.org/api.html) 。

#### redis.sha1hex

对字符串执行SHA1算法

例子：

```
127.0.0.1:6379> eval 'return redis.sha1hex(ARGV[1])' 0 "foo"
"0beec7b5ea3f0fdbc95d0dd47f3c5bc275da8a33"
```

---

### Emitting Redis logs from scripts

在 Lua 脚本中，可以通过调用 redis.log 函数来写 Redis 日志(log)：

```
redis.log(loglevel, message)
```

其中， message 参数是一个字符串，而 loglevel 参数可以是以下任意一个值：
- redis.LOG_DEBUG
- redis.LOG_VERBOSE
- redis.LOG_NOTICE
- redis.LOG_WARNING

上面的这些等级(level)和标准 Redis 日志的等级相对应。

对于脚本散发(emit)的日志，只有那些和当前 Redis 实例所设置的日志等级相同或更高级的日志才会被散发。

以下是一个日志示例：

```
redis.log(redis.LOG_WARNING,"Something is wrong with this script.")
```

执行上面的函数会产生这样的信息：

```
[32343] 22 Mar 15:21:39 # Something is wrong with this script.
```

---

### Sandbox and maximum execution time

脚本永远不应该尝试访问外部系统，如文件系统或任何其他系统调用。脚本应该只对 Redis 数据和传递的参数进行操作。

脚本也受到最大执行时间限制（默认值为 5 秒）。这个默认超时时间很大，因为脚本通常应该在毫秒内运行完成。限制主要值为了处理开发过程中创建的意外无限循环。

可以通过 redis.conf 或使用 CONFIG GET/CONFIG SET 命令以毫秒精度修改脚本的可执行最大时间。影响最大执行时间的配置参数为 `lua-time-limit`。

当脚本达到超时时间时，他不会被 Redis 自动终止，因为 Redis 必须保证脚本执行的原子性，而这违反了 Redis 与脚本引擎之间的约定。中断脚本意味着可能会留下写了一半的数据集。出于这个原因，当脚本执行时间超过指定事件时，会发生以下情况下：
- Redis 记录脚本运行时间过长。
- Redis 开始重新接受其他客户端的命令请求，但只有 [SCRIPT KILL](script-kill.md) 和 `SHUTDOWN NOSAVE` 两个命令会被处理，对于其他命令请求，Redis 服务器只是简单地返回 BUSY 错误。
- 可以使用 [SCRIPT KILL](script-kill.md) 命令将一个仅执行只读命令的脚本杀死，因为只读命令并不修改数据，因此杀死这个脚本不会破坏数据的完整性。
- 如果脚本已经执行过写命令，那么唯一允许执行的操作就是 `SHUTDOWN NOSAVE`，它通过停止服务器来阻止当前数据集写入磁盘。

---

### EVALSHA in the context of pipelining

在流水线请求的上下文中使用 [EVALSHA](evalsha.md) 命令时，要特别小心，因为在流水线中，必须保证命令的执行顺序。

一旦流水线中因为 [EVALSHA](evalsha.md) 命令而发生了 NOSCRIPT 错误，那么这个流水线就再也没有办法重新执行了，否则的话，命令的执行顺序就会被打乱。

为了防止出现以上所说的问题，客户端库实现应该实施以下的其中一项措施：

- 总是在流水线中使用 [EVAL](eval.md) 命令。
- 检查流水线中要用到的所有命令，找到其中的 [EVAL](eval.md) 命令，并使用 [SCRIPT EXISTS](script-exists.md) 命令检查要使用的脚本是不是都全部已经保存在缓存里面了。如果所需的全部脚本都可以在缓存里找到，那么就可以放心地将所有 [EVAL](eval.md) 命令改成 [EVALSHA](evalsha.md) 命令，否则的话，就要在流水线的顶端(top)将缺少的脚本用 [SCRIPT LOAD](script-load.md) 命令加上去。

---

### Debugging Lua scripts

从 Redis 3.2 开始，Redis 支持本地 Lua 调试。Redis Lua debugger 是一个远程调试器，由服务器（Redis本身）和客户端（默认为 redis-cli）组成。

Lua 调试器在 Redis 文档的 [Lua scripts debugging](../topics/ldb.md) 部分进行了描述。