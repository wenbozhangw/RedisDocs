## DISCARD

    起始版本：2.0.0
    
刷新一个 [transaction](../topics/transactions.md) 中所有在排队等待的指令，并且将连接回复到正常状态。

如果已使用 [WATCH](watch.md)，[DISCARD](discard.md) 将释放所有被 [WATCH](watch.md) 的 key。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) : always OK. 