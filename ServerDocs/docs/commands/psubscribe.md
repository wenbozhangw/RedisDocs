## PSUBSCRIBE pattern [pattern ...]

    起始版本：2.0.0.
    时间复杂度：O(N)，其中 N 是客户端已订阅的 pattern 数量。

订阅给定的模式(patterns)。

支持的模式(patterns)有：

- h?llo subscribes to hello, hallo and hxllo
- h*llo subscribes to hllo and heeeello
- h[ae]llo subscribes to hello and hallo, but not hillo

如果要逐字匹配特殊字符串，请使用 `\` 转义它们。