## SETBIT key offset value

    起始版本：2.2.0.
    时间复杂度：O(1)。

设置或清空 key 对应的字符串值在 _offset_ 处的 bit。

根据值可以设置或清除该 bit，该值可以是 0 或 1。

当 key 不存在时，会创建一个新的字符串值。要确保这个字符串大到在 _offset_ 出有 bit 值。参数 _offset_ 需要大于等于 0，并且小于 2<sup>32</sup> (限制 bitmap 大小为 512MB)。当 _key_ 对应的字符串增大的时候，新增的部分 bit 值都是设置为 0。

**警告：** 当设置最后一个 bit (_offset_ 等于 2<sup>32</sup> - 1)并且 key 还没有一个字符串值或者其值是一个较小的字符串时，Redis 需要立即分配所有内存，这有可能会导致服务阻塞一会儿。在一台 2010 MacBook Pro 上，offset 为 2<sup>32</sup> - 1（分配 512MB）需要 ~300ms，offset 为 2<sup>30</sup> - 1（分配 128M）需要 ~80ms，offset 为 2<sup>28</sup> - 1（分配32MB）需要 ~30ms，offset为 2<sup>26</sup> - 1（分配 8MB）需要 ~8ms。注意，一旦第一次内存分配完，后面对同一个 key 调用 [SETBIT](SETBIT.md) 就不需要提前内存分配。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)：存储在 _offset_ 处的原始 bit。

---

### Examples

```
redis> SETBIT mykey 7 1
(integer) 0
redis> SETBIT mykey 7 0
(integer) 1
redis> GET mykey
"\u0000"
redis> 
```

---

### Pattern: accessing the entire bitmap

在某些情况下，您需要一次设置单个 bitmap 的所有 bit，例如将其初始化为默认的非零 bit 值时。可以通过多次调用 [SETBIT](SETBIT.md) 命令来做到这一点，需要设置的每一个 bit 调用一次。但是，作为优化，您可以使用单个 [SET](SET.md) 命令来设置整个 bitmap。

bitmap 不是实际的数据类型，而是在 String 类型上定义的一组面向 bit 的操作（有关更多信息，请参阅 [Bitmaps section of the Data Types Introduction page](../topics/data-types-intro.md#bitmaps) ）。这意味着 bitmap 可以与字符串命令一起使用，最重要的是与 [SET](SET.md) 和 [GET](GET.md) 一起使用。

因为 Redis 的字符串是二进制安全的，bitmap 被简单的编码为字节流(bytes stream)。字符串的第一个字节对应于 bitmap 的 _offset_ 0..7，第二个字节对应 8..15 范围，依次类推。

例如，设置几个 bit 后，获取 bitmap 的字符串值将如下所示：

```
> SETBIT bitmapsarestrings 2 1
> SETBIT bitmapsarestrings 3 1
> SETBIT bitmapsarestrings 5 1
> SETBIT bitmapsarestrings 10 1
> SETBIT bitmapsarestrings 11 1
> SETBIT bitmapsarestrings 14 1
> GET bitmapsarestrings
"42"
```

通过获取 bitmap 的字符串表示，客户端可以通过在其本地编程语言中使用本地 bit 操作提取 bit 值来解析响应的字节。对称的，也可以在客户端执行 bits-to-bytes 编码并使用结果字符串调用 [SET](SET.md) 来设置整个 bitmap。

---

### Pattern: setting multiple bits

[SETBIT](SETBIT.md) 擅长设置单个 bit，需要设置多个 bit 时可以多次调用。要优化此操作，您可以将多个 [SETBIT](SETBIT.md) 调用替换为对应可变参数 [BITFIELD](BITFIELD.md) 命令的单个调用以及 `u1` 类型字段的使用。

例如，上面的示例可以替换为：

```
> BITFIELD bitsinabitmap SET u1 2 1 SET u1 3 1 SET u1 5 1 SET u1 10 1 SET u1 11 1 SET u1 14 1
```

---

### Advanced Pattern: accessing bitmap ranges

也可以使用 [GETRANGE](GETRANGE.md) 和 [SETRANGE](SETRANGE.md) 字符串命令来有效地访问 bitmap 中的 bit offsets 范围。一下是可以使用 [EVAL](EVAL.md) 命令允许的常用 Redis Lua 脚本中的示例实现：

```
--[[
Sets a bitmap range

Bitmaps are stored as Strings in Redis. A range spans one or more bytes,
so we can call [SETRANGE](/commands/setrange) when entire bytes need to be set instead of flipping
individual bits. Also, to avoid multiple internal memory allocations in
Redis, we traverse in reverse.
Expected input:
  KEYS[1] - bitfield key
  ARGV[1] - start offset (0-based, inclusive)
  ARGV[2] - end offset (same, should be bigger than start, no error checking)
  ARGV[3] - value (should be 0 or 1, no error checking)
]]--

-- A helper function to stringify a binary string to semi-binary format
local function tobits(str)
  local r = ''
  for i = 1, string.len(str) do
    local c = string.byte(str, i)
    local b = ' '
    for j = 0, 7 do
      b = tostring(bit.band(c, 1)) .. b
      c = bit.rshift(c, 1)
    end
    r = r .. b
  end
  return r
end

-- Main
local k = KEYS[1]
local s, e, v = tonumber(ARGV[1]), tonumber(ARGV[2]), tonumber(ARGV[3])

-- First treat the dangling bits in the last byte
local ms, me = s % 8, (e + 1) % 8
if me > 0 then
  local t = math.max(e - me + 1, s)
  for i = e, t, -1 do
    redis.call('SETBIT', k, i, v)
  end
  e = t
end

-- Then the danglings in the first byte
if ms > 0 then
  local t = math.min(s - ms + 7, e)
  for i = s, t, 1 do
    redis.call('SETBIT', k, i, v)
  end
  s = t + 1
end

-- Set a range accordingly, if at all
local rs, re = s / 8, (e + 1) / 8
local rl = re - rs
if rl > 0 then
  local b = '\255'
  if 0 == v then
    b = '\0'
  end
  redis.call('SETRANGE', k, rs, string.rep(b, rl))
end
```

**注意：** 从 bitmap 中获取一系列 bit offset 的实现留给读者作为练习。
