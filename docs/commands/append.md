## APPEND key value

    起始版本：2.0.0.
    时间复杂度：O(1)。均摊时间复杂度是 O(1)，因为 redis 用的动态字符串的库在每次分配空间的时候回增加一倍的可用空闲空间，所以在添加的 value 较小而且已经存在的 value 是任意大小的情况下，均摊时间复杂度是 O(1)。

如果 key 已经存在，并且值为字符串，那么这个命令会把 value 追加到原来值（value）的结尾。如果 key 不存在，那么它将首先创建一个空字符串的 key，在执行追加操作，这种情况 [APPEND](append.md) 将类似于 [SET](set.md) 操作。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers) ： 返回 append 后字符串（value）的长度。

---

### Examples

```
redis> EXISTS mykey
(integer) 0
redis> APPEND mykey "Hello"
(integer) 5
redis> APPEND mykey " World"
(integer) 11
redis> GET mykey
"Hello World"
redis> 
```

---

### Pattern: Time series

[APPEND](append.md) 命令可以用来连接一系列固定长度的样例，与使用列表相比这样更加紧凑。通常会用来记录 _节拍序列(time series)_。

没收到一个新的节拍样例就可以这样记录：

```
APPEND timeseries "fixed-size sample"
```

在节拍序列里，可以很容易地访问序列中的每个元素：
- [STRLEN](strlen.md) 可以用来计算样例个数。
- [GETRANGE](getrange.md) 允许随机访问序列中的各个元素。如果序列中有明确的节拍信息，在 Redis 2.6 中就可以使用 [GETRANGE](getrange.md) 配合 Lua 脚本来实现一个二分查找算法。
- [SETRANGE](setrange.md) 可以用来覆写已有的节拍序列。

该模式的局限在于只能做追加操作。Redis 目前缺少裁剪字符串的命令，所以无法方便地把序列裁剪成指定的大小。但是，节拍序列在空间占用上效率极好。

提示：可以根据当前的 Unix 时间切换到不同的 key，这样每个 key 的样本量可能相对较少，避免处理非常大的 key，并使这种模式更加友好地分布在多个 Redis 实例中。

使用固定大小的字符串（在实际实现中使用二进制格式更好）对传感器的温度进行采样的实例。

```
redis> APPEND ts "0043"
(integer) 4
redis> APPEND ts "0035"
(integer) 8
redis> GETRANGE ts 0 3
"0043"
redis> GETRANGE ts 4 7
"0035"
redis> 
```