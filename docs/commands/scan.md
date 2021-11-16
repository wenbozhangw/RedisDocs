## SCAN cursor [MATCH pattern] [COUNT count] [TYPE type]

    起始版本：2.8.0。
    时间复杂度：每次调用为O(1)。O(N) 用于完整的迭代，包括足够的命令调用使游标返回到 0。N 是集合内的元素数。

[SCAN](scan.md) 命令及其相关的 [SSCAN](sscan.md)、[HSCAN](hscan.md) 和 [ZSCAN](zscan.md) 都用于增量迭代一个集合元素。
- [SCAN](scan.md) 命令用于迭代当前数据库中的 key 集合。
- [SSCAN](sscan.md) 命令用于迭代 Set 类型的元素。
- [HSCAN](hscan.md) 命令用于迭代 Hash 类型中的键值对。
- [ZSCAN](zscan.md) 命令用于迭代 Sorted Set 集合中的元素和元素对应的分值。

以上列出的命令都支持增量迭代，它们每次执行都只会返回少量元素，所以这些命令可以用于生产环境，而不会出现像 [KEYS](keys.md) 或 [SMEMBERS](smembers.md) 命令带来的可能会阻塞服务器的问题。

不过，[SMEMBERS](smembers.md) 命令可以返回集合 key 当前包含的所有元素，但是对于 SCAN 这类增量式迭代来说，有可能在增量迭代过程中，集合元素被修改，对返回值无法提供完全准确的保证。

因为 [SCAN](scan.md)、[SSCAN](sscan.md)、[HSCAN](hscan.md) 和 [ZSCAN](zscan.md) 四个命令的工作方式都非常类似，所以这个文档会一并介绍这四个命令，需要注意的是 [SSCAN](sscan.md)、[HSCAN](hscan.md) 和 [ZSCAN](zscan.md) 命令的第一个参数总是 key；[SCAN](scan.md) 命令则不需要在第一个参数提供任何 key，因为它迭代的是当前数据库中的所有key。

---

### SCAN basic usage

SCAN 命令是一个基于游标(cursor)的迭代器。这意味着每次被调用都需要使用上一次这个调用返回的游标作为该次调用的游标参数，以此来延续之前的迭代过程。

当 SCAN 命令的 cursor 参数被设置为 0 时，服务器将开始一次新的迭代，而当服务器向用户返回值为 0 的游标时，表示迭代已结束。一下是一个 SCAN 命令的迭代过程：

```
redis 127.0.0.1:6379> scan 0
1) "17"
2)  1) "key:12"
    2) "key:8"
    3) "key:4"
    4) "key:14"
    5) "key:16"
    6) "key:17"
    7) "key:15"
    8) "key:10"
    9) "key:3"
   10) "key:7"
   11) "key:1"
redis 127.0.0.1:6379> scan 17
1) "0"
2) 1) "key:5"
   2) "key:18"
   3) "key:0"
   4) "key:2"
   5) "key:19"
   6) "key:13"
   7) "key:6"
   8) "key:9"
   9) "key:11"
```

在上面这个例子中，第一次迭代使用 0 作为游标，表示开始一次新的迭代。第二次迭代使用的是第一次迭代时返回的游标 17，作为新的迭代参数。

显而易见， **SCAN命令的返回值**是包含两个元素的数组，第一个数组元素是用于进行下一次迭代的新游标，而第二个数组元素则是一个数组，这个数组中包含了所有被迭代的元素。

在第二次调用 SCAN 命令时，命令返回了游标 0，这表示迭代已经结束，整个数据集已经被完整遍历过了。以 0 作为游标开始一次新的迭代，一直调用 [SCAN](scan.md) 命令，直到命令返回游标 0，我们成这个过程为一次 **full iteration**。

---

### Scan guarantees

[SCAN](scan.md) 命令以及其他 [SCAN](scan.md) 家族的命令，在进行完整的遍历的情况下可以为用户带来与完整迭代相关的保证。
- 从完整遍历开始到完整遍历结束期间，一直存在于数据集内的所有元素都会被完整遍历返回；这意味着，如果有一个元素，它从遍历开始直到遍历结束期间都存在与被遍历的数据集当中，那么 [SCAN](scan.md) 命令总会在某次迭代中将这个元素返回给用户。
- 完整迭代永远不会返回从完整迭代开始到结束的集合中不存在的任何元素。因此，如果一个元素在迭代开始之前被删除，并且在迭代持续的所有时间里都不会被添加回集合，[SCAN](scan.md) 确保永远不会返回该元素。

然而，因为 [SCAN](scan.md) 关联的状态很少（只有游标），所以它有以下缺点：
- 同一个元素可能会被返回多次。处理重复元素的工作交由应用程序负责，比如说，可以考虑将迭代返回的元素仅仅用于可以安全地重复执行多次的操作上。
- 如果一个元素是在迭代过程中被添加到数据集的，又或者是在迭代过程中从数据集中被删除的，那么这个元素可能会被返回，也可能不会。

---

### Number of elements returned at every SCAN call

[SCAN](scan.md) 系列函数不保证每次调用返回的元素数在给定范围内。这些命令也可能返回 0 个元素，只要返回的游标不为 0，客户端就不应该认为迭代完成。

不过命令返回的元素数量总是符合一定规则的，对于一个大数据集来说，SCAN 命令每次最多可能会返回数十个元素；而对于一个小集合来说，如果这个数据集的底层表示为编码数据结构（小的 sets, hashes and sorted sets），那么增量迭代命令将在一次调用中返回数据集中的所有元素。

如果需要的话，用户可以通过增量式迭代命令提供的 **COUNT** 选项来指定每次迭代返回元素的最大值。

---

### The COUNT option

虽然 [SCAN](scan.md) 不保证每次迭代所返回的元素数量，我们可以使用 **COUNT** 选项，对命令的行为进行一定程度上的调整。COUNT 选项的作用就是让用户告知迭代命令，在每次迭代中应该从数据集里返回多少元素。使用 COUNT 选项对于 SCAN 命令相当于是一种提示，大多数情况下，这种是都可以有效地控制返回的数量。
- COUNT 参数的默认值是 10。
- 当迭代键空间，或者一个Set、Hash、Sorted Set 的数据集比较大时，如果没有使用 **MATCH** 选项，那么命令返回的元素数量通常和 COUNT 选项指定的数量稍多一些。请查看后面的文档 ： why SCAN may return all the elements at once
- 当迭代编码为 intsets 的 Sets（仅由整数组成的小集合）或编码为 ziplists 的 Hash 和 Sorted Set 时（由不同值构成的一个小哈希或者一个小有序集合），通常所有元素都在第一个 [SCAN](scan.md) 调用中返回，而不管 COUNT 的值。

注意：**并非每次迭代都要使用相同的 COUNT 值**，用户可以在每次迭代中按自己的需要随意改变 COUNT 值，只要记得将上次迭代返回的游标用到下次迭代里面就可以了。

---

### The MATCH option

类似于将模式作为唯一参数的 [KEYS](keys.md) 命令的行为，SCAN 命令可以只迭代匹配给定 glob-style 模式的元素。

为此，只需要在 [SCAN](scan.md) 命令的末尾追加 `MATCH <pattern>`参数（它适用于所有 SCAN 系列命令）。

以下是一个使用 **MATCH** 迭代的示例：

```
redis 127.0.0.1:6379> sadd myset 1 2 3 foo foobar feelsgood
(integer) 6
redis 127.0.0.1:6379> sscan myset 0 match f*
1) "0"
2) 1) "foo"
   2) "feelsgood"
   3) "foobar"
redis 127.0.0.1:6379>
```

重要的是要注意 **MATCH** 过滤器是在从集合中检索元素之后，在数据返回给客户端之前的这段时间进行的。这意味着如果模式匹配集合中的元素很少，则 [SCAN](scan.md) 命令或许会在多次执行中都不返回任何元素。

以下是这种情况的一个例子：

```
redis 127.0.0.1:6379> scan 0 MATCH *11*
1) "288"
2) 1) "key:911"
redis 127.0.0.1:6379> scan 288 MATCH *11*
1) "224"
2) (empty list or set)
redis 127.0.0.1:6379> scan 224 MATCH *11*
1) "80"
2) (empty list or set)
redis 127.0.0.1:6379> scan 80 MATCH *11*
1) "176"
2) (empty list or set)
redis 127.0.0.1:6379> scan 176 MATCH *11* COUNT 1000
1) "0"
2)  1) "key:611"
    2) "key:711"
    3) "key:118"
    4) "key:117"
    5) "key:311"
    6) "key:112"
    7) "key:111"
    8) "key:110"
    9) "key:113"
   10) "key:211"
   11) "key:411"
   12) "key:115"
   13) "key:116"
   14) "key:114"
   15) "key:119"
   16) "key:811"
   17) "key:511"
   18) "key:11"
redis 127.0.0.1:6379>
```

可以看出，以上的大部分迭代都不返回任何元素。在最后一次迭代，我们通过将 COUNT 选项的参数设置为 1000，强制命令为本次迭代扫描更多元素，从而使得命令返回的元素也变多了。

---

### The TYPE option

从 6.0 版本开始，您可以使用此选项要求 [SCAN](scan.md) 仅返回匹配给定类型的对象，允许您遍历数据库以查找特定类型的 key。**TYPE** 选项仅适用于全库 [SCAN](scan.md)，不适用于 [HSCAN](hscan.md) 或 [ZSCAN](zscan.md) 等。

type 参数与 [TYPE](type.md) 命令返回的字符串名称相同。请注意一些 Redis 类型（例如 GeoHashes、HyperLogLogs、Bitmaps 和 Bitfields）可能在内部使用其他 Redis 类型（例如 string 或 zset）实现，因此无法通过 [SCAN](scan.md) 将其与相同类型的其他 key 区分开来。

```
redis 127.0.0.1:6379> GEOADD geokey 0 0 value
(integer) 1
redis 127.0.0.1:6379> ZADD zkey 1000 value
(integer) 1
redis 127.0.0.1:6379> TYPE geokey
zset
redis 127.0.0.1:6379> TYPE zkey
zset
redis 127.0.0.1:6379> SCAN 0 TYPE zset
1) "0"
2) 1) "geokey"
   2) "zkey"
```

需要注意的是，**TYPE** 过滤器也会在从数据库检索元素后执行，因此该选项不会减少服务器完成完整迭代所需的工作量，对于稀少的类型，您可能在多次迭代中都不返回任何元素。

---

### Multiple parallel iterations

在同一时间，可以有任意多个客户端对同一数据集进行迭代，客户端每次执行迭代都需要传入一个游标，并在迭代执行之后获得一个新的游标，而这个游标就包含了迭代的所有状态，因此，服务器无须为迭代记录任何状态。

---

### Terminating iterations in the middle

因为迭代的所有状态都是保存在游标里面，而服务器无须为迭代保存任何状态，所以客户端可以在中途停止一个迭代，而无须对服务器进行任何通知。即使有任意数量的迭代在中途停止，也不会产生任何问题。

---

### Calling SCAN with corrupted cursor

使用 [SCAN](scan.md) 命令传入间断(broken)、负数、超出范围或者其他非正常的游标来执行增量式迭代并不会造成服务器崩溃，但可能会让命令产生未定义的行为。未定义的行为指的是，[SCAN](scan.md) 命令对返回值无法做出保证。

只有两种游标是合法的：
- 在开始一个新的迭代时，游标必须为 0 。
- SCAN 执行后返回的，用于延续迭代过程的游标。

---

### Guarantee of termination

[SCAN](scan.md) 命令所使用的算法只保证在数据集的大小有界的情况下，迭代才会停止，换句话说，如果被迭代数据集的大小不断地增长的话，[SCAN](scan.md) 命令可能永远也无法完成一次完整迭代。

从直觉上可以看出，当一个数据集不断变大的时候，想要访问这个数据集中的所有元素就需要做越来越多的工作，能否结束一个迭代取决于用户执行[SCAN](scan.md) 迭代的速度是否比数据集增长的速度更快。

---

### Why SCAN may return all the items of an aggregate data type in a single call?

在 COUNT 选项的文档中，我们声明，有时 [SCAN](scan.md) 这一系列命令可能会在一次调用中一次返回 Set、Hash 或 Sorted Set 的所有元素，而不管 COUNT 选项的值是多少。发生这种情况的原因是，只有当我们正在扫描的聚合数据类型表示为哈希表时，基于游标的迭代器可以实现，并且是有用的。然而，Redis 使用 [memory optimization](../topics/memory-optimization.md) ，其中小的聚合数据类型，在它们达到给定的项目数量或单个元素的最大大小之前，会使用 compact single-allocation packed encoding。在这种情况下， [SCAN](scan.md) 没有有意义的游标返回，并且必须一次迭代整个数据结构，因此他唯一合理的行为是在调用中返回所有内容。

然而，一旦数据结构更大并被提升为使用真正的 hash table，[SCAN](scan.md) 命令系列将resort 正常行为。请注意，由于这种返回所有元素的特殊行为仅适用于小型聚合，因此他对命令复杂性或延迟没有影响。然而，转换为真实哈希表的确切限制是 [user configurable]() ，因此您可以在单个调用中看到的最大元素数取决于决和数据类型可以有多大并且仍然使用 packed 表示。

另请注意，此命令特定于 [SSCAN](sscan.md)、[HSCAN](hscan.md) 和 [ZSCAN](zscan.md)。 [SCAN](scan.md) 本身没有此行为，因为键空间始终由哈希表表示。

---

### Return Value

[SCAN](scan.md)、[SSCAN](sscan.md)、[HSCAN](hscan.md) 和 [ZSCAN](zscan.md) 命令都返回一个包含两个元素的 multi-bulk reply：回复的第一个元素是字符串表示的无符号 64 位整数（cursor），恢复的第二个元素是另一个 multi-bulk reply，包含了本次被迭代的元素。
- [SCAN](scan.md) 命令返回的每个元素都是一个 key。
- [SSCAN](sscan.md) 命令返回的每个元素都是一个集合成员。
- [HSCAN](hscan.md) 命令返回的每个元素都是一个键值对，一个键值对由一个键和一个值组成。
- [ZSCAN](zscan.md) 命令返回的每个元素都是一个有序集合元素，一个有序集合元素由一个成员（member）和一个分值（score）组成。

---

### History

- &gt;= 6.0: 提供 [TYPE](type.md) 子命令。

---

### Additional examples

迭代 hash 中的键值对：

```
redis 127.0.0.1:6379> hmset hash name Jack age 33
OK
redis 127.0.0.1:6379> hscan hash 0
1) "0"
2) 1) "name"
   2) "Jack"
   3) "age"
   4) "33"
```