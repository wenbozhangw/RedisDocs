## SETEX key seconds value

    起始版本：2.0.0。
    时间复杂度：O(1)。

设置 key 的值为指定的字符串，并且设置 key 在给定的 seconds 时间之后超时过期。这个命令等效于执行下面的命令：
```
SET mykey value
EXPIRE mykey seconds
```

[SETEX](SETEX.md) 是原子的，也可以通过把上面两个命令放到 [MULTI](MULTI.md)/[EXEC](EXEC.md) 块中执行的方式实现。相比连续执行上面两个命令，它更快，因为当 Redis 作为缓存使用时，这个操作更加常用。

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)

### Examples

```
redis> SETEX mykey 10 "Hello"
"OK"
redis> TTL mykey
(integer) 10
redis> GET mykey
"Hello"
redis> 
```