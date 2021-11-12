## PING [message]

    起始版本：1.0.0。

如果未提供参数，则返回 `PONG`，否则以批量形式返回参数的副本。此命令通常用于测试连接是否仍然有效，或者延迟检测。

如果客户端订阅了 channel 或 pattern，它将改为返回第一个位置为 `PONG`，而第二个位置为 empty bulk，除非提供了参数，在这种情况下，他返回一个参数副本。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings)

---

### Examples

```
redis> PING
"PONG"
redis> PING "hello world"
"hello world"
redis> 
```