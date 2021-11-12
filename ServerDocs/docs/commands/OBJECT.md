## OBJECT subcommand [arguments [arguments ...]]

    起始版本：2.2.3.
    时间复杂度：O(1)，对于所有当前实现的子命令。

[OBJECT](OBJECT.md) 命令允许检查与 key 关联的 Redis 对象的内部结构。这对于调试或了解您的 key 是否使用特殊编码的数据类型以节省空间很有用。当使用 Redis 作为缓存时，您的应用程序可以使用 [OBJECT](OBJECT.md) 命令报告的信息来实现应用层的 key 的驱逐策略。

[OBJECT](OBJECT.md) 支持多个子命令：
- `OBJECT REFCOUNT <key>`：返回与指定 key 关联的值的引用数，此命令主要用于调试。
- `OBJECT ENCODING <key>`：返回用于存储于 key 关联的值的内部表示类型。
- `OBJECT IDLETIME <key>`：返回指定 key 对应的 value 自从被存储之后空闲的时间，以秒为单位（没有读写操作的请求），这个值返回 10 秒为单位的秒级别时间，但在未来的实现中可能会有所不同。当 `maxmemory-policy` 设置为 `LRU` 时，此子命令可用。
- `OBJECT FREQ <key>`：返回存储在指定 key 上的对象的对数(logarithmic)访问频率计数器。当 `maxmemory-policy` 设置为 `LRU` 时，此子命令可用。
- `OBJECT HELP`：返回一个简洁地帮助文本。

Objects 可以用不同的方式编码：
- string 可以编码为 raw（普通字符串编码）或 int（表示 64 位有符号间隔中的整数的字符串，以这种方式编码以节省空间）。
- list 可以编码为 ziplist 或 linkedlist。ziplist 是用于为小列表节省空间的特殊表示。
- set 可以编码为 intset 或 hashtable。intset 是一种特殊编码，用于仅有整数组成的小集合。
- Hash 可以编码为 ziplist 或 hashtable。ziplist 是一种用于小 hash 的特殊编码。
- Sorted set 可以编码为 ziplist 或 skiplist 格式。对于 List 类型的小 sorted set 可以使用 ziplist 进行特殊编码，而 skiplist 编码适用于任何大小的 sorted set。

一旦您执行了使 Redis 无法保留节省空间的编码的操作，所有特殊编码的类型都会自动转换为通用类型。

---

### Return Value

不同的子命令会对应不同的返回值。
- 子命令 `refcount` 和 `idletime` 返回整数。
- 子命令 `encoding` 返回 bulk reply。

如果你尝试检查的参数丢失，将会返回空。

---

### Examples

```
redis> lpush mylist "Hello World"
(integer) 4
redis> object refcount mylist
(integer) 1
redis> object encoding mylist
"ziplist"
redis> object idletime mylist
(integer) 10
```

接下来的例子你可以看到 Redis 一旦不能够使用节省编码类型时，编码方式的改变。

```
redis> set foo 1000
OK
redis> object encoding foo
"int"
redis> append foo bar
(integer) 7
redis> get foo
"1000bar"
redis> object encoding foo
"raw"
```