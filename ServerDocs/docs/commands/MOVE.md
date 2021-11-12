## MOVE key db

    起始版本：1.0.0.
    时间复杂度：O(1)。

将当前选择的数据库（见 [SELECT](select.md) ）的 key 移动到给定的目标数据库当中。当 key 在目标数据库中已经存在时，它什么都不做。因此，可以使用 [MOVE](MOVE.md) 最为锁(locking)原语(primitive)。

---

### Return Value

[Integer reply](../topics/protocol.md#resp-integers)，特定的：
- 移动成功返回 1
- 失败则返回 0
