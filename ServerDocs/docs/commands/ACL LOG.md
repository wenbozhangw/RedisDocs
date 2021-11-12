## ACL LOG

    自 6.0.0 起可用。
    时间复杂度: 0(N)。其中 N 是显示的条目数。

该命令显示最近的 ACL 安全事件列表：
1. 无法通过 [AUTH](AUTH.md) 或 [HELLO](HELLO.md) 验证其连接。
2. 命令被拒绝，因为违反当前 ACL 规则。
3. 命令被拒绝，因为当前 ACL 规则中不允许访问 key。

可选参数指定要显示的条目数。默认最多返回是个失败。指定的 [RESET](RESET.md) 参数清除日志。从最近的条目开始显示。

---

### Return Value

当被调用以显示安全事件时：

[Array reply](../topics/protocol.md#resp-arrays) : ACL 安全事件列表。

当使用 [RESET](RESET.md) 调用时：

[Simple string reply](../topics/protocol.md#resp-simple-strings) : 如果安全日志已清除，返回 OK 。

---

### Examples

```
> AUTH someuser wrongpassword
(error) WRONGPASS invalid username-password pair
> ACL LOG 1
1)  1) "count"
    2) (integer) 1
    3) "reason"
    4) "auth"
    5) "context"
    6) "toplevel"
    7) "object"
    8) "AUTH"
    9) "username"
   10) "someuser"
   11) "age-seconds"
   12) "4.0960000000000001"
   13) "client-info"
   14) "id=6 addr=127.0.0.1:63026 fd=8 name= age=9 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=48 qbuf-free=32720 obl=0 oll=0 omem=0 events=r cmd=auth user=default"
```