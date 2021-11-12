## INCRBYFLOAT key increment

    起始版本：2.6.0.
    时间复杂度：O(1).

将 key 对应的字符串表示的浮点数值增加 `increment` 后存储。通过使用负数 `increment`，结果是存储在 key 上的值减少（通过加法的明显属性）。如果该键不存在，则在执行操作前先将其设置为 0。如果出现下列情况之一，则返回错误：
- key 包含错误类型的值（非字符串）。
- 当前的 key 或者相加后的值不能解析为一个双精度浮点值。（超出精度范围了）

如果命令操作成功，相加后的值将替换原值存储在对应的 key 上，并以 string 的类型返回。

已经包含在字符串 key 和 `increment` 参数中的值都可以选择以指数表示法提供，但是在 `increment` 计算之后的值也以相同的格式存储，即一个整数（如果需要）后跟一个点，以表示该数字的小数部分的可变数字。后面的零总是被删除的。

输出的精度固定在小数点后的 17 位，而不管计算的实际内部精度如何。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)：当前 key 加上 `increment` 之后的值。

---

### Examples

```
redis> SET mykey 10.50
"OK"
redis> INCRBYFLOAT mykey 0.1
"10.6"
redis> INCRBYFLOAT mykey -5
"5.6"
redis> SET mykey 5.0e3
"OK"
redis> INCRBYFLOAT mykey 2.0e2
"5200"
redis> 
```

---

### Implementation details

该命令总是作为 [SET](SET.md) 操作在副本连接(replication link)和 Append Only 文件中传播，因此底层浮点数学实现中的差异不会成为不一致的来源。
