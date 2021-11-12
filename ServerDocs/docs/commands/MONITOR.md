## MONITOR

    起始版本：1.0.0。

[MONITOR](MONITOR.md) 是一个 debugger 命令，返回服务器处理的每一个命令，他能帮助我们了解在数据库上发生了什么操作，可以通过 `redis-cli` 和 `telnet` 命令使用。

在将 Redis 用作数据库和分布式缓存系统时，查看服务器处理的所有请求的能力对于发现应用程序中的错误很有用。

```
$ redis-cli monitor
1339518083.107412 [0 127.0.0.1:60866] "keys" "*"
1339518087.877697 [0 127.0.0.1:60866] "dbsize"
1339518090.420270 [0 127.0.0.1:60866] "set" "x" "6"
1339518096.506257 [0 127.0.0.1:60866] "get" "x"
1339518099.363765 [0 127.0.0.1:60866] "eval" "return redis.call('set','x','7')" "0"
1339518100.363799 [0 lua] "set" "x" "7"
1339518100.544926 [0 127.0.0.1:60866] "del" "x"
```

使用 `SIGINT`(Ctrl-C) 停止通过 redis-cli 运行的 [MONITOR](MONITOR.md) 返回的输出。

```
$ telnet localhost 6379
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
MONITOR
+OK
+1339518083.107412 [0 127.0.0.1:60866] "keys" "*"
+1339518087.877697 [0 127.0.0.1:60866] "dbsize"
+1339518090.420270 [0 127.0.0.1:60866] "set" "x" "6"
+1339518096.506257 [0 127.0.0.1:60866] "get" "x"
+1339518099.363765 [0 127.0.0.1:60866] "del" "x"
+1339518100.544926 [0 127.0.0.1:60866] "get" "x"
QUIT
+OK
Connection closed by foreign host.
```
使用 [QUIT](quit.md) 或 [RESET](RESET.md) 命令可以停止通过 telnet 命令使用 [MONITOR](MONITOR.md) 返回的输出。

---

### Commands not logged by MONITOR

出于安全考虑，[MONITOR](MONITOR.md) 的输出不会记录任何管理命令，并且敏感数据会在命令 [AUTH](AUTH.md) 中进行编辑。

此外，命令[QUIT](quit.md) 也不会被记录。

---

### Cost of running MONITOR

由于 [MONITOR](MONITOR.md) 命令返回服务器处理的所有的命令，所以在性能上会有一些消耗。

在不运行 [MONITOR](MONITOR.md) 命令的情况下，benchmark 的测试结果：

```
$ src/redis-benchmark -c 10 -n 100000 -q
PING_INLINE: 101936.80 requests per second
PING_BULK: 102880.66 requests per second
SET: 95419.85 requests per second
GET: 104275.29 requests per second
INCR: 93283.58 requests per second
```

在运行 [MONITOR](MONITOR.md) 命令的情况下，benchmark 的测试结果(redis-cli monitor > /dev/null) ：

```
$ src/redis-benchmark -c 10 -n 100000 -q
PING_INLINE: 58479.53 requests per second
PING_BULK: 59136.61 requests per second
SET: 41823.50 requests per second
GET: 45330.91 requests per second
INCR: 41771.09 requests per second
```

在这种特定的情况下，运行一个 [MONITOR](MONITOR.md) 命令能够降低 50% 的吞吐量，运行多个 [MONITOR](MONITOR.md) 命令降低的吞吐量更多。

---

### Return Value

**非标准返回值**，无限的返回服务器端处理的命令流。

---

### History

- &gt;= 6.0: [AUTH](AUTH.md) 从命令的输出中排除。
- &gt;= 6.2: 可以调用 [RESET](RESET.md) 来退出监控模式。
- &gt;= 6.2.4: [AUTH](AUTH.md), [HELLO](HELLO.md), [EVAL](EVAL.md), EVAL_RO, [EVALSHA](EVALSHA.md) 和 EVALSHA_RO 包含在命令的输出中。