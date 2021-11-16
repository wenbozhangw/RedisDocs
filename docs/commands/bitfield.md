## BITFIELD key [GET type offset] [SET type offset value] [INCRBY type offset increment] [OVERFLOW WRAP|SAT|FAIL]

    起始版本：3.2.0.
    时间复杂度：对于每个指定的子命令 O(1)。

该命令将 Redis 字符串视为 bit 数组，并且能够处理不同 bit widths 和任意非（必要）对齐偏移量的特定整数字段。实际上，可以使用该命令对一个有符号的 5 位整型数的 1234 位设置指定值，也可以对一个 31 位无符号整型数的 4567 位进行取值。类似地，在堆指定的整数进行自增和自减操作，本命令可以提供有保证的、可配置的上溢(overflow)和下溢(underflow)处理操作。

[BITFIELD](bitfield.md) 能够在同一个命令调用中使用多个 bit fields 进行操作。它需要一个要执行的操作列表，并返回一个 array reply，其中每个数组都与参数列表中的相应操作相匹配。

例如，一下命令在 bit offset 100 处递增一个 5 bit 无符号整数，并在 bit offset 0 处获取 4 bit 无符号整数的值：

```
> BITFIELD mykey INCRBY i5 100 1 GET u4 0
1) (integer) 1
2) (integer) 0
```

提示：
1. 用 GET 指令对超出当前字符串长度的 bit（含 key 不存在的情况）进行寻址，执行操作的结果会对缺失部分的 bits 赋值为 0。
2. 用 SET 或 INCRBY 指令对超出当前字符串长度的 bit（含 key 不存在的情况）进行寻址，将会扩展字符串并对扩展部分进行补 0，扩展方式包括：按需扩展、按最小长度扩展和按最大寻址能力扩展。

---

### Supported subcommands and integer types

以下是支持的命令列表。
- **GET** \<type\> \<offset\> —— 返回指定的 bit field。
- **SET** \<type\> \<offset\> \<value\> —— 设置指定 bit field 的值并返回它的原始值。
- **INCRBY** \<type\> \<offset\> \<increment\> —— 自增或自减（如果 increment 为负数）指定 bit field 的值并返回它的新值。

还有一个命令通过设置溢出行为来改变调用 INCRBY 指令的后续操作：
- **OVERFLOW** [WRAP|SAT|FAIL]

当需要一个整型时，有符号整型需要在位数前加 `i`，无符号在位数前加 `u`。例如，u8 是一个 8 位无符号整型，i16 是一个 16 位的有符号整型。

有符号整型最大支持 64 位，而无符号整型最大支持 63 位。对无符号整型的限制，是由于当前 Redis 协议不能在响应消息中返回 64 位无符号整数。

---

### Bits and positional offsets

bitfield 命令有两种方式来指定 bit offset。如果指定了没有任何前缀的数字，则它仅用作字符串内基于零的 bit offset。

但是，如果 offset 以 # 字符作为前缀，则指定的 offset 将乘以整数类型的宽度，例如：

```
BITFIELD mystring SET i8 #0 100 SET i8 #1 200
```

将在 offset 0 处设置第一个 i8 整数，在 offset 8 处设置第二个整数。这样，如果您想要的是给定大小的普通整数数组，则不必在客户端内部进行数学计算。

---

### Overflow control

使用 OVERFLOW 命令，用户可以通过指定下列其中一种行为来调整自增或自减操作溢出（或下溢）后的行为：
- **WRAP**：回环算法，适用于有符号和无符号整型两种类型。对于无符号整型，回环计数将对整型最大值进行取模操作（C语言的标准行为）。对于有符号整型，上溢从最负的负数开始取数，下溢则从最大的正数开始取数，例如，如果 i8 整型的值设为 127 ，自加 1 后的值变为 -128。
- **SAT**：饱和算法，下溢之后设置为最小的整型值，上溢之后设置为最大的整数值。例如，i8 整型的值从 120 开始加 10 后，结果是 127，继续增加，结果还是保持为 127。下溢也是同理，但量结果值将会保持在最负的负数值。
- **FAIL**：失败算法，这种模式下，在检测到上溢或下溢时，不做任何操作。相应的返回值会设置为 NULL，并返回给调用者。

注意每种溢出（OVERFLOW）控制方法，仅影响紧跟在 INCRBY 命令后的子命令，知道重新指定溢出（OVERFLOW）控制方法。

如果没有指定溢出控制方法，默认情况下，将使用 **WARP** 算法。

```
> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 1
2) (integer) 1
> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 2
2) (integer) 2
> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 3
2) (integer) 3
> BITFIELD mykey incrby u2 100 1 OVERFLOW SAT incrby u2 102 1
1) (integer) 0
2) (integer) 3
```

---

### Return Value

本命令返回一个针对子命令给定位置的处理结果组成的数组。OVERFLOW 子命令在响应消息中，不会统计结果的条数。

下面是 OVERFLOW FAIL 返回 NULL 的样例：

```
> BITFIELD mykey OVERFLOW FAIL incrby u2 102 1
1) (nil)
```

---

### Motivations

本命令的动机(motivations)是为了能够在单个 large bitmap 中高效地存储多个小整数（或分割几个键以避免使用大键），同时开放 Redis 提供的新使用案例，尤其是在实时分析领域。这种使用案例可以通过指定的溢出控制方法来支持。

有趣的试试：Reddit 的 2017 年愚人节项目 [r/place](https://reddit.com/r/place) 是 [built using the Redis BITFIELD command](https://redditblog.com/2017/04/13/how-we-built-rplace/) ，以便在内存中表示协作画布。

---

### Performance considerations

通常，[BITFIELD](bitfield.md) 是一个非常快的命令，但是注意，对端字符串的远地址(fat bits)寻址，将会比在存在的 bit 执行命令更加耗时。

---

### Orders of bits

[BITFIELD](bitfield.md) 命令使用的 bitmap 表现形式，可看做是从 0 位开始的，例如：把一个 5 位的无符号整数 23，对一个所有位事先置 0 的 bitmap，从第 7 位开始复制，其结果如下所示：

```
+--------+--------+
|00000001|01110000|
+--------+--------+
```

当 offset 和 integer size 是字节边界对齐时，此时与大端模式（big endian）相同，但是，当字节边界未对齐时，那么理解字节顺序将变得非常重要。