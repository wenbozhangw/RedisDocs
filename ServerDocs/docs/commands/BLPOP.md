## BLPOP key [key ...] timeout

    自 2.0.0 起可用。
    时间复杂度：O(N)，其中 N 为提供的 key 个数。

[BLPOP](BLPOP.md) 是阻塞式列表的弹出原语。它是 [LPOP](LPOP.md) 的阻塞版本，这是因为当给定的列表内没有任何元素可供弹出的时候，连接将被 [BLPOP](BLPOP.md) 命令阻塞。当给定多个 key 参数时，按参数 key 的先后顺序依次检查各个列表，弹出第一个非空列表的头元素。

---

### Non-blocking behavior

当 [BLPOP](BLPOP.md) 被调用时，如果给定 key 内至少有一个非空列表，那么弹出遇到的第一个非空列表的头元素，并和被弹出元素所属的列表的 key 一起，组成结果返回给调用者。

当存在多个给定 key 时，该命令按给定 key 参数排列的先后顺序，依次检查各个列表。我们假设 key list1 不存在，而 list2 和 list3 都是非空列表。考虑以下命令：

```
BLPOP list1 list2 list3 0
```

[BLPOP](BLPOP.md) 保证返回一个存在于 list2 里的元素（因为它是从 list1 -> list2 -> list3 这个顺序的第一个非空列表）。

---

### Blocking behavior

如果指定的 key 都不存在， [BLPOP](BLPOP.md) 将阻塞连接，直到有另一个客户端对给定的这些 key 的任意一个执行 [LPUSH](LPUSH.md) 或 [RPUSH](RPUSH.md)  命令。

一旦有新的数据出现在其中一个列表里，那么这个命令会解除阻塞状态，并且返回 key 和弹出的元素值。

当 [BLPOP](BLPOP.md) 命令引起客户端阻塞并且设置了一个非零的超时参数 timeout 时，如果经过了指定的 timeout 时间内仍没有出现一个针对特定 key 的 push 操作，则客户端会解除阻塞状态并返回一个 nil 的 multi-bulk value。

**timeout 参数被作为 double 值，指定要阻塞的最大秒数。** timeout 0 可用于无限期的阻塞。

---

### What key is served first ? What client ? What element ? Priority ordering details.

- 当客户端为多个 key 尝试阻塞时，若至少存在一个 key 拥有元素，纳闷返回的键值对就是从左到右数第一个拥有一个或多个元素的 key。在这种情况下，客户端不会被阻塞。比如对于这个例子 `BLPOP key1 key2 key3 key4 0`，假设 key2 和 key4 都非空，那么就会返回 key2 里的一个元素。
- 当多个客户端为同一个 key 阻塞的时候，第一个被处理的客户端是等待最长时间的那个（即第一个因为该 key 而阻塞的客户端）。一旦一个客户端解除阻塞，那么他就不会保持任何优先级，当它因为下一个 [BLPOP](BLPOP.md) 命令而再次被阻塞时，会在处理完那些被同一个 key 阻塞的客户端后才处理它（即从第一个被阻塞的处理到最后一个被阻塞的）。
- 当一个客户端同时被多个 key 阻塞时，如果多个 key 的元素同时可用（可能是因为事务或者某个 Lua 脚本想多个 list 添加元素），那么客户端会解除阻塞，并使用第一个接收到 push 操作的 key（假设它拥有足够的元素为我们的客户端服务，因为有可能存在其他客户端同样是被这个 key 阻塞着）。从根本上来说，在执行完每个命令之后，Redis 会把一个所有 key 都获得数据并且至少使一个客户端阻塞了的 list 运行一次。这个 list 按照新数据的接收时间进行整理，即是从第一个接受数据的 key 到最后一个。在处理每个 key 的时候，只要这个 key 里有元素，Redis 就会对所有等待这个 key 的客户端按照 "先进先出" (FIFO) 的顺序进行服务。如果这个 key 是空的，或者没有客户端在等待这个 key，那么将会去处理下一个从之前的命令或事务或脚本中获得新数据的 key，如此等等。

---

### Behavior of BLPOP when multiple elements are pushed inside a list.

有时候一个 list 会在同一个概念的命令的情况下接收到多个元素：

- 像 `LPUSH mylist a b c` 这样的可变 push 操作。
- 在堆一个想同一个 list 进行多次 push 操作的 [MULTI](MULTI.md) 块执行完 [EXEC](EXEC.md)  语句后。
- 使用 Redis 2.6 或以上的版本执行一个 Lua 脚本。

当多个元素被 push 进入一个被客户端阻塞着的 list 的时候，Redis 2.4 和 Redis 2.6 或以上的版本所采取的行为是不一样的。

对于 Redis 2.6 来说，所采取的行为是先执行多个 push 命令，然后在执行了这个命令之后，再去为被阻塞的客户端提供服务。看看下面命令顺序。

```
Client A: BLPOP foo 0
Client B: LPUSH foo a b c
```

如果上面的情况是发生在 Redis 2.6 或以上的服务器上，客户端 A 会接收到 c 元素，因为在 [LPUSH](LPUSH.md) 命令执行后，list 包含了 c,b,a 这三个元素，所以从左边取第一个元素就会返回 c。

相反，Redis 2.4 是以不同的方式工作的：客户端会在 push 操作的上下中提供服务，所以当 `LPUSH foo a b c` 开始向 list 中 push 第一个元素，他就被传送给客户端 A，也就是客户端 A 会接收到 a （第一个被 push 的元素）。

Redis 2.4 的这种行为会在复制或者持续把数据存入 AOF 文件的时候引发很多问题，所以为了防止这些问题，很多更一般性的、并且在语义上更简单的行为被引入到 Redis 2.6 中。

需要注意的是，一个 Lua 脚本或者一个 [MULTI](MULTI.md)/[EXEC](EXEC.md) 块可能会 push 一堆元素进入一个 list 后，再删除这个 list。在这种情况下，被阻塞的客户端完全不会被提供服务，并且只要在执行某个单一命令、事务或者脚本后 list 中没有出现元素，他就会被继续阻塞下去。

---

### BLPOP inside a MULTI/EXEC transaction

[BLPOP](BLPOP.md) 可用于 pipeline（发送多个命令并且批量读取回复），特别是当它是 pipeline 里的最后一个命令的时候，这种设定更加有意义。

在一个 [MULTI](MULTI.md) / [EXEC](EXEC.md) 块里使用 [BLPOP](BLPOP.md) 并没有很大的意义，因为他要求整个服务器被阻塞以保证块执行时的原子性，这就阻止了其他客户的执行一个 push 操作。因此，一个在 [MULTI](MULTI.md) / [EXEC](EXEC.md) 里面的 [BLPOP](BLPOP.md) 命令会在 list 为空的时候返回一个 nil 值，这跟超时(timeout)的时候发生的一样。

如果你喜欢科幻小说，那么想象一下时间是以无限的速度在 [MULTI](MULTI.md) / [EXEC](EXEC.md) 块中流逝...

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays) : 具体来说 : 
- 当没有元素返回的时，会弹出一个 nil 的 multi-bulk value，并且 timeout 过期。
- 当有元素弹出时，会返回一个 wo-element multi-bulk 值，其中第一个元素是弹出元素的 key，第二个元素是 value。

---

### History

- &gt;= 6.0 : timeout 被改为 double 而不是 integer。

---

### Examples

```
redis> DEL list1 list2
(integer) 0
redis> RPUSH list1 a b c
(integer) 3
redis> BLPOP list1 list2 0
1) "list1"
2) "a"
```

---

### Reliable queues

当 [BLPOP](BLPOP.md) 返回一个元素给客户端的时候，他也从 list 中把该元素移除。这意味着该元素就只存在于客户端的上下文中：如果客户端在处理这个返回元素的过程崩溃了，那么这个元素就永远丢失了。

在一些我们希望是更可靠的消息传递系统中的应用上，这可能会导致一些问题。这种时候，请查看 [BRPOPLPUSH](BRPOPLPUSH.md) 命令，这是 [BLPOP](BLPOP.md) 的一个变形，他会在把元素传给客户端之前先把该元素加入到一个目标 list 中。

---

### Pattern: Event notification

用来阻塞 list 的操作有可能是不同的阻塞原语。比如在某些应用里，你也许会为了等待新元素进入 Redis Set 而阻塞队列，直到有个新元素加入到 Set 中，这样就可以在不轮询的情况下获得元素。这就要求要有一个 [SPOP](SPOP.md) 的阻塞版本，而这事实上并不可用。但是我们可以通过阻塞 list 操作轻易完成这个任务。

消费者会做的：
```
LOOP forever
    WHILE SPOP(key) returns elements
        ... process elements ...
    END
    BRPOP helper_key
END
```

而在生产者角度我们可以这样简单地使用：
```
MULTI
SADD key element
LPUSH helper_key x
EXEC
```