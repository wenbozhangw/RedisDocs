## Redis Protocol specification

Redis 客户端使用成为 **RESP** (REdis Serialization Protocol) 的协议与 Redis 服务器进行通信。虽然该协议是专为 Redis 设计的，但它也可用于其他 client-server 软件项目。

RESP 是一下各项之间的折中：

- Simple to implement.
- Fast to parse.
- Human readable.

RESP 可以徐磊好不同的数据类型，如 integers、strings、arrays。还有一种特定的错误类型。请求以字符串数组的形式从客户端发送到 Redis 服务器，这些字符串表示要执行的命令的参数。Redis 使用特定命令 (command-specific) 的数据类型进行回复。

RESP 是二进制安全的 (binary-safe)，不需要处理从一个进程传输另一个进程的批量数据，因为它使用前缀长度 (prefixed-length) 来传输批量数据。

注意：此处概述的协议仅用于 client-server 通信。Redis Cluster 使用不同的二进制协议赖在节点之间交换消息。

### Networking layer

客户端连接到 Redis 服务器，创建一个 TCP 连接到端口 6379。

虽然 RESP 在技术上是非 TCP 特定的 (non-TCP specific)，但在 Redis 的上下文中，该协议仅用于 TCP 连接（或等效的面向流的连接，如 Unix 套接字）。

### Request-Response model

Redis 接受由不同参数组成的命令。收到命令后，将对其进行处理并将回复发送回客户端。

这是最简单的模型，但有两个例外：

- Redis 支持 pipelining (本文档稍后介绍)。所以客户端可以一次发送多个命令，然后等待回复。
- 当 Redis 客户端订阅 Pub/Sub 通道 (channel) 时，协议语义发生变化，变成 *push* 协议，即客户端不再需要发送命令，因为服务器在收到新消息会自动向客户端发送（对订阅的客户端通道）。

除去以上两个协议，Redis 协议是一个简单的 request-response protocol。

### RESP protocol description

RESP 协议是在 Redis 1.2 中引入的，但它在 Redis 2.0 中成为了与 Redis 服务器通信的标准方式。这是您应该在 Redis 客户端中实现的协议。

RESP 实际上是一种序列化协议，它支持以下数据类型：Simple Strings, Errors, Integers, Bulk Strings and Arrays。

RESP 在 Redis 中作为 request-response 协议使用的方式如下：

- 客户端将命令作为批量字符串的 RESP 数组发送到 Redis 服务器。
- 服务器根据命令实现以其中一种 RESP 类型进行回复。

在 RESP 中，某些数据的类型取决于第一个字节：

- 对于 **Simple Strings**，回复的第一个字节是 "+"
- 对于 **Errors**，回复的第一个字节是 "-"
- 对于 **Integers**，回复的第一个字节是 ":"
- 对于 **Bulk Strings**，回复的第一个字节是 "$"
- 对于 **Arrays**，回复的第一个字节是 "*"

此外，RESP 能够使用后面指定的 Bulk Strings 或 Arrays 的特殊变体来表示 Null 值。在 RESP 中，协议的不同部分总是以 "\r\n" (CRLF) 终止。

---

#### RESP Simple Strings

Simple Strings 按以下方式编码：加号字符 (a plus character)，后跟不能包含 CR 或 LF 字符（不允许换行）的字符串，以 CRLF 结尾（即"\r\n"）。

Simple Strings 用于以最小开销传输非二进制安全字符。例如，许多 Redis 命令在成功时回复 "OK"，作为 RESP 简单字符串使用以下 5 个字节进行编码：

    "+OK\r\n"

为了发送二进制安全字符串，使用 RESP Bulk Strings代替。
当 Redis 回复一个简单字符串时，客户端应该向调用者返回一个字符串，改字符串由 "+" 之后的第一个字符组成，知道字符串的末尾，不包括最后的 CRLF 字节。

---

#### RESP Errors

RESP 具有特定的错误数据类型。实际上错误与 RESP 简单字符串完全一样，但第一个字符是减号 '-' 字符而不是加号。RESP 中 Simple Strings 和 Errors 的真正区别在于客户端将错误视为异常，而构成 Error 类型的字符串就是错误消息本身。

基本格式为：

    "-Error message\r\n"

错误回复仅在发生错误时发送，例如，如果您尝试对错误的数据类型执行操作，或者命令不存在等。当收到错误回复时，library 客户端应引发异常。

以下是错误回复的实例：

    -ERR unknown command 'foobar'
    -WRONGTYPE Operation against a key holding the wrong kind of value

"-"之后的第一个单词，知道第一个空格或换行符，表示返回的错误类型。这只是 Redis 使用的约定，不是 RESP 错误格式的一部分。

例如，ERR 是一般错误，而 WRONGTYPE 是更具体的错误，它暗示客户端视图对错误的数据类型执行操作。这称为**错误前缀** (**Error Prefix**)，是一种允许客户端了解服务器返回的错误类型的方法，而无需依赖给定的确切消息，该消息可能会随着时间而改变。

客户端实现可以针对不同的错误返回不同类型的异常，或者可以通过将错误名称作为字符串直接提供给调用者来提供捕获错误的通用方法。

然而，这样的特性不应该被认为是至关重要的，因为它很少有用，并且优先的客户端实现可能只是返回一个通用的错误条件，比如 false。

---

#### RESP Integers

这种类型只是一个 CRLF 终止的字符串，代表一个整数，以 ":" 字节为前缀。例如 ":0\r\n" 或 ":1000\r\n" 是整数回复。

[comment]: <> (todo link)

许多 Redis 命令返回 RESP 整数，例如 [INCR]() 、 [LLEN]() 和 [LASTSAVE]() 。

返回的整数没有特殊含义，它只是 [INCR]() 的增量数字， [LASTAVE]() 的 UNIX 时间等等。但是，返回的整数保证在有符号的 64 位整数范围内。整数回复也被广泛用于返回 true or false。例如，像 [EXISTS]() 或 [SISMEMBER]() 这样的命令将返回 1 表示 true，0 表示 false。

如果操作实际执行，其他命令如 [SADD]() 、 [SERM]() 和 [SETNX]() 将返回 1，否则返回 0。

以下命令将回复整数回复：
[SETNX]()、
[DEL]()、
[EXISTS]()、
[INCR]()、
[INCRBY]()、
[DECR]()、
[DECRBY]()、
[DBSIZE]()、
[LASTSAVE]()、
[RENAMENX]()、
[MOVE]()、
[LLEN]()、
[SADD]()、
[SREM]()、
[SISMEMBER]()、
[SCARD]() 。

---

#### RESP Bulk Strings

Bulk Strings 用于表示长度最大为 512MB 的单个二进制安全字符串。批量字符串按一下方式编码：

- "$" 字节后跟组成字符串的字节数（前缀长度, a prefixed length），以 CRLF 结尾。
- 实际的字符串数据。
- 最后 CRLF。

所以字符串 "foobar" 的编码如下：

    "$6\r\nfoobar\r\n"

当一个空字符串是：

    "$0\r\n\r\n"

还可以使用 RESP Bulk Strings，使用一种用于表示 Null 值的特殊格式来表示值不存在。在这种特殊格式中，长度为 -1，并且没有数据，因此 Null 表示为：

    "$-1\r\n"

<div style="display: none;">

#### Null Bulk String

</div>

这称为 <span id="nullBulkString">**Null Bulk String** </span>。

当服务器使用 Null Bulk String 回复时，客户端库 API 不应返回空字符串，而是返回 nil 对象。例如，Ruby 库应该返回 "nil"，而 C 库应返回 NULL（或在回复对象中设置特殊标志），等等。

---

#### RESP Arrays

客户端使用 RESP 数组向 Redis 服务器发送命令。类似地，某些 Redis 命令将元素集合返回给客户端使用 RESP 数组作为回复类型。一个例子是返回列表元素的 [LRANGE]() 命令。

RESP Arrays使用以下格式发送：

- 一个 * 字符作为第一个字节，然后是数组中元素的数量作为十进制数，然后是 CRLF。
- Array 的每个元素的附加 RESP 类型。

所以一个空数组如下：

    "*0\r\n"

两个 RESP Bulk Strings "foo" 和 "bar" 的数组被编码为：

    "*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"

正如您在数组前缀的 *&lt;count&gt;CRLF 部分之后看到的那样，组成数组的其他数据类型只是一个接一个地连接在一起。例如，三个整数的数组编码如下：

    "*3\r\n:1\r\n:2\r\n:3\r\n"

数组可以包含混合类型，元素不必是相同类型。例如，一个包含四个整数的列表和一个批量字符串可以编码如下：

    *5\r\n
    :1\r\n
    :2\r\n
    :3\r\n
    :4\r\n
    $6\r\n
    foobar\r\n

（为了清晰起见，回复被分成多行）。

服务器发送的第一行是 *5\r\n 以指定后面将有五个回复。然后发送构成多批量回复项目的每个回复。

Null Array的概念也存在，它是指定 Null 值的另一种方式（通常使用 Null Bulk String，但由于历史原因，我们有两种格式）。

例如，当 [BLPOP](../commands/blpop.md) 命令超时时，它会返回一个计数为 -1 的空数组，如下例所示：

    "*-1\r\n"

当 Redis 用 Null Array 回复时，客户端库 API 应返回一个 null 对象而不是一个空的 Array。这是区分空列表和不同条件（例如 [BLPOP](../commands/blpop.md) 命令的超时条件）所必须的。

在 RESP 中可以使用数组嵌套数组。例如，两个数组嵌套的编码如下：

    *2\r\n
    *3\r\n
    :1\r\n
    :2\r\n
    :3\r\n
    *2\r\n
    +Foo\r\n
    -Bar\r\n

（为了清晰起见，回复被分成多行）。

上面的 RESP 数据类型编码了一个有两个数组元素的数组，该数组由一个包含三个整数 1、2、3的数组和一个简单字符串和一个错误的数组组成。

---

#### Null elements in Arrays

Array 的单个元素可能为 Null。这用于 Redis 回复中，以表示这些元素丢失而不是空字符串。当缺少指定的 key 时，当与 GET 模式选项一起使用时，SORT 命令可能会发生这种情况。包含 Null 元素的 Array 回复示例 ：

    *3\r\n
    $3\r\n
    foo\r\n
    $-1\r\n
    $3\r\n
    bar\r\n

第二个元素是 Null。客户端库应该返回如下内容：

    ["foo", nil, "bar"]

请注意，这不是前几节所说的例外，而只是进一步指示协议的示例。

---

### Sending commands to a Redis Server

既然您已经书序 RESP 序列化格式，那么编写 Redis 客户端库的实现就很容易了。我们可以进一步指定客户端和服务器之间的交互是如何工作的：

- 客户端向 Redis 服务器发送一个仅包含大容量字符串的 RESP 数组。
- Redis 服务器响应客户端发送任何有效 RESP 数据类型作为应答。

因此，例如典型的交互可能如下。

客户端发送命令 LLEN mylist 以获取存储在密钥 mylist 中的列表的长度，服务器以整数响应作为回复，如下例所示（ C : 是客户端, S  : 服务器）。

    C: *2\r\n
    C: $4\r\n
    C: LLEN\r\n
    C: $6\r\n
    C: mylist\r\n

    S : :48293\r\n

向往常一样，为了简单起见，我们用换行符分割协议的不同部分，但实际的交互是客户端发送 "*2\r\n$4\r\nLLEN\r\n$6\r\nmylist\r\n" 作为一个整体。

### Multiple commands and pipelining

客户端可以使用相同的连接来发出多个命令。支持 Pipelining，因此客户端可以通过一个写操作发送多个命令，而不需要在发出下一个命令之前读取前一个命令的服务器响应。所有的响应都可以在最后阅读。

有关更多信息，请查看 [page about Pipelining]() 。

### Inline Commands

有时您手上只有 telnet，您需要发送一个命令到 Redis 服务器。虽然 Redis 协议易于实现，但在交互式会话中使用并不理想，并且 redis-cli 可能并不总是可用。处于这个原因，Redis 还以一种专为人类设计的特殊方式接受命令，称为 **inline command** 格式。

一下是使用 inline command 的服务器/客户端聊天实例（C : 是客户端, S : 服务器）

    C: PING
    S: +PONG

以下是返回整数的 inline command 另一个示例：

    C: EXISTS somekey
    S: :0

基本上，您只需在 telnet 会话中编写以空格分隔的参数。由于没有命令以 * 开头，而是在同一请求协议中使用，Redis 能够检测到这种情况并解析您的命令。

### High performance parser for the Redis protocol

虽然 Redis 协议非常易读且易于实现，但它可以以类似于二进制协议的性能来实现。

RESP 使用前缀长度来传输批量数据，因此永远不需要像使用 JSON 那样扫描负载以寻找特殊字符，也不需要引用需要发送到服务器的负载。

Bulk 和 Multi Bulk 长度可以使用对每个字节执行单个操作同时扫描 CR 字符的代码进行处理，如下面的 C 代码：

```c
#include <stdio.h>

int main(void) {
    unsigned char *p = "$123\r\n";
    int len = 0;
    
    p++;
    while(*p != '\r') {
        len = (len * 10) + (*p - '0');
        p++;
    }
    
    /* Now p points at '\r', and the len is in bulk_len. */
    printf("%d\n", len);
    return 0;
}
```

识别出第一个 CR 后，它可以与后面的 LF 一起跳过，无需任何处理。然后可以使用不以任何方式检查有效负载的单个读取操作读取批量数据。最后剩下的 CR 和 LF 字符被丢弃而不做任何处理。

虽然在性能上可与二进制协议相媲美，但 Redis 协议在大多数非常高级的语言中实现起来要简单得多，从而减少了客户端软件中的错误数量。