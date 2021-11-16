## ZSCAN key cursor [MATCH pattern] [COUNT count]

    起始版本：2.8.0。
    时间复杂度：对于每次调用 O(1)。O(N) 用于完整的迭代，包括足够的命令调用使游标返回到 0。N 是集合内的元素数。

有关 [ZSCAN](zscan.md) 文档，请参阅 [SCAN](scan.md) 。