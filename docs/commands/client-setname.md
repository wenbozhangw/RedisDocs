## CLIENT SETNAME connection-name

    起始版本：2.6.9。
    时间复杂度：O(1)。

[CLIENT SETNAME](client-setname.md) 为当前连接分配一个名字。

这个名字会显示在 [CLIENT LIST](client-list.md) 命令的结果中，用于识别当前正在与服务器进行连接的客户端。

举个例子，在使用 Redis 构建队列(queue) 时，客户根据连接负责的角色(role)，为信息生产者(producer) 和信息消费者(consumer) 分布设置不同的名字。

名字使用 Redis 的字符串类型来保存，最大可以占用 512MB。另外，为了避免和 [CLIENT LIST](client-list.md) 命令的输出格式发生冲突，名字里不允许使用空格。

要移除一个连接的名字，可以将连接的名字设置为空字符串，这不是有效的连接名称，因为它用于此特定目的。

使用 [CLIENT GETNAME](client-getname.md) 命令可以取出连接的名字。

新创建的默认连接是没有名字的。

提示：在 Redis 应用程序发生连接泄露时，为连接设置名字是一种很好的 debug 手段。

---

### Return Value

[Simple string reply](../topics/protocol.md#resp-simple-strings) : 连接名称设置成功返回 OK。