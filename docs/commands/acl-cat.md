## ACL CAT [categoryname]

    自 6.0.0 起可用。
    时间复杂度: O(1)因为 categories 和 commands 是固定集。

如果不带参数调用，该命令将显示可用的 ACL 类别。如果给出了类别名称，该命令将显示指定类别中的所有 Redis 命令。

ACL 类别对于创建一次包含或排除大量命令的 ACL 规则非常有用，而无需指定每个命令。例如，一下规则将让用户 karin 执行除可能影响服务器稳定性的最危险操作之外的所有操作：

```
ACL SETUSER karin on +@all -@dangerous
```

我们首先将所有命令添加到 karin 能够执行的命令集中，然后我们删除所有 dangerous 命令。

检查所有可用类别非常简单：

```
> ACL CAT
 1) "keyspace"
 2) "read"
 3) "write"
 4) "set"
 5) "sortedset"
 6) "list"
 7) "hash"
 8) "string"
 9) "bitmap"
10) "hyperloglog"
11) "geo"
12) "stream"
13) "pubsub"
14) "admin"
15) "fast"
16) "slow"
17) "blocking"
18) "dangerous"
19) "connection"
20) "transaction"
21) "scripting"
```

然后我们可能想知道哪些命令属于给定类别的一部分：

```
> ACL CAT dangerous
 1) "flushdb"
 2) "acl"
 3) "slowlog"
 4) "debug"
 5) "role"
 6) "keys"
 7) "pfselftest"
 8) "client"
 9) "bgrewriteaof"
10) "replicaof"
11) "monitor"
12) "restore-asking"
13) "latency"
14) "replconf"
15) "pfdebug"
16) "bgsave"
17) "sync"
18) "config"
19) "flushall"
20) "cluster"
21) "info"
22) "lastsave"
23) "slaveof"
24) "swapdb"
25) "module"
26) "restore"
27) "migrate"
28) "save"
29) "shutdown"
30) "psync"
31) "sort"
```

---

### Return Value

[Array reply](../topics/protocol.md#RESP Arrays) : ACL 类别列表或给定类别的命令列表。如果将无效的类别名称作为参数给出，该命令可能会返回错误。