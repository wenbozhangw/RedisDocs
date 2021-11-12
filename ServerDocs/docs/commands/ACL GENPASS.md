## ACL GENPASS [bits]

    自 6.0.0 起可用。
    时间复杂度: O(1)。

ACL 用户需要一个可靠的密码才能在没有安全风险的情况下对服务器进行身份验证。这样的密码不需要人类记住，只需要计算机记住，因此它可以非常长和强大（外部攻击者无法猜测）。如果可用，[ACL GENPASS](ACL GENPASS.md) 命令从 /dev/urandom 开始生成密码，否则（在没有 /dev/urandom 的系统中）它使用较弱的系统，这可能仍然比手动选择弱密码更好。

默认情况下（如果 /dev/urandom 可用）密码很强，可以在 Redis 应用程序的上下文中用于其他用途，例如为了创建唯一的会话标识符或其他类型的不可猜测且不冲突的 ID。密码生成也非常简易，因为我们实际上并没有在每次执行时访问 /dev/urandom 的位。Redis 启动时使用 /dev/urandom 创建一个 seed，然后它将在计数器模式下使用 SHA256，以 HMAC-SHA256 (seed, counter) 作为原语，已创建更多的随机字节。这意味着应用程序开发人员应该可以随意滥用 [ACL GENPASS](ACL GENPASS.md) 来根据需要创建尽可能多的安全伪随机字符串。

命令输出是二进制字符串的十六进制表示。默认情况下，它发出 256 位 (即 64 个十六进制字符)。用户可以提供从 1 到 1024 的比特数形式的参数，以改变输出长度。请注意，所提供的位数始终四舍五入为下一个 4 个倍数。因此，例如，仅要求 1 bits 密码将导致以单个十六进制字符的形式发出 4 位的密码。

---

### Return value

[Bulk string reply](../topics/protocol.md#RESP Bulk Strings) : 默认情况下 64 字节字符串表示 256 位伪随机数据。否则，如果需要参数，则输出字符串长度为指定的位数（四舍五入到下一个 4 的倍数）除以 4。

---

### Examples

```
> ACL GENPASS
"dd721260bfe1b3d9601e7fbab36de6d04e2e67b0ef1c53de59d45950db0dd3cc"

> ACL GENPASS 32
"355ef3dd"

> ACL GENPASS 5
"90"
```