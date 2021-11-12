## SUBSCRIBE channel [channel ...]

    起始版本：2.0.0.
    时间复杂度：O(N) 其中 N 是要订阅的频道数量。

客户端订阅给定频道的信息。

一旦客户端进入订阅状态，除了额外的 [SUBSCRIBE](subscribe.md)、[PSUBSCRIBE](psubscribe.md)、[UNSUBSCRIBE](unsubscribe.md)、[PUNSUBSCRIBE](punsubscribe.md)、[PING](ping.md)、[RESET](RESET.md) 和 [QUIT](quit.md) 命令之外，它不应发出任何其他命令。

---

### History

- &gt;= 6.2：可以调用 [RESET](RESET.md) 退出订阅状态。

