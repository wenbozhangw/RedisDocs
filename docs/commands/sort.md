## SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]

    起始版本：1.0.0.
    时间复杂度：O(N+M*log(M))，其中 N 是列表或集合中要排序的元素数量，M 是返回元素的数量。当元素未排序时，复杂度为 O(N)。

返回或存储包含在指定 key 对应的 [list](../topics/data-types.md#lists), [set](../topics/data-types.md#sets) 或 [sorted set](../topics/data-types.md#sorted-sets) 中的元素。

从 Redis 7.0.0 开始，还有这个命令的 `SORT_RO` 只读变体。

默认情况下，按照数值类型进行排序，元素通过它们的值进行比较，值为双精度浮点数形式。这是最简单的形式：

```
SORT mylist
```

假设 `mylist` 是一个数字列表，此命令将返回相同的列表，其中元素从小到大排序。为了将数字从大到小排序，请使用过 `DESC` 修饰符：

```
SORT mylist DESC
```

当 `mylist` 包含字符串值并且您想按字典(lexicographically)顺序对它们进行排序时，请使用 `ALPHA` 修饰符。

```
SORT mylist ALPHA
```

假设您正确设置了 `!LC-COLLATE` 环境变量，Redis 支持 UTF-8。

返回元素的数量可以通过 `LIMIT` 修饰符限制。此修饰符有一个 `offset` 参数，指定了跳过的元素数量；还带有一个 `count` 参数，指定了从 offset 开始返回的元素数量。下面的例子将会返回排序后的列表 `mylist` 从第 0 个元素（offset是从 0 开始的）开始的10个元素：

```
SORT mylist LIMIT 0 10
```

几乎所有的修饰符可以一起使用。下面的例子将返回按字典顺序降序排序后的 5 个元素：

```
SORT mylist LIMIT 0 5 ALPHA DESC
```

---

### Sorting by external keys

有时您想使用外部 key 作为权重对元素进行排序以进行比较，而不是使用 list、set 或 sorted set 中的实际元素。假设列表 `mylist` 包含元素 1、2、3，分别代表了存储在 object_1、object_2、object_3 中的对象的唯一 ID。当这些对象关联到存储在 weight_1、weight_2、weight_3 中的权重后，[SORT](sort.md) 命令就能使用这些权重按照下述语句来对 `mylist` 排序：

```
SORT mylist BY weight_*
```

`BY` 选项带有一个模式（在本例中等于 weight_*），用于生产用于排序的 key。这些 key 名称是通过将第一次出现的 * 替换为列表中元素的实际值（在本例中为 1、2、3）而获得的。

---

### SKip sorting the elements

`BY` 选项还可以是一个并不存在的 key，这会导致 [SORT](sort.md) 命令跳过排序操作。如果您想在没有排序开销的情况下检索外部 key（参见下面的 GET 选项），这很有用。

```
SORT mylist BY nosort
```

---

### Retrieving external keys

我们前面的例子只返回排序后的 ID。在某些情况下，获取实际的对象而不是它们的 ID 更加重要（object_1、object_2 和 object_3）。获取存储在一个 list、set 或 sorted set 中的 key 可以使用以下命令：

```
SORT mylist BY weight_* GET object_*
```

`GET` 选项可以多次使用，以便为原始 list、set 或 sorted set 的每个元素获取更多的 key。

还可以通过使用特殊 `#` 模式获取 `GET` 元素本身：

```
SORT mylist BY weight_* GET object_* GET #
```

---

### Storing the result of a SORT operation

默认情况下，[SORT](sort.md) 命令返回排序后的元素给客户端。使用 `SOTRE` 选项，可以将结果存储在一个特定的列表中，以代替返回到客户端。

```
SORT mylist BY weight_* STORE resultkey
```

`SORT ... STORE` 的一种有趣应用模式，是联合 [EXPIRE](expire.md) 超时命令返回 key，以便在应用中可以缓存 [SORT](sort.md) 操作的返回结果。其他客户端将会使用已缓存的类别，代替每个请求的 [SORT](sort.md) 调用。当 key 即将过期时，一个更新版本的缓存将会通过 `SORT ... STORE` 再次创建。

注意，为了正确实现这种模式，很重要的一点是防止多个客户端同时重建缓存。此时需要使用一些锁（具体的示例 [SETNX](setnx.md) ）。

---

### Using hashes in BY and GET

可以在 hash 的属性上按下述语法使用 `BY` 和 `GET` 选项：

```
SORT mylist BY weight_*->fieldname GET object_*->fieldname
```

字符串 `->` 用于区分 key 名称和 hash field 的名称。key 被替换为上面所记录的，结果 key 中存储的 hash 用于获取特定 hash 的 field。

---

### Return Value

[Array reply](../topics/protocol.md#resp-arrays)：在不传递 `store` 选项的情况下，该命令返回已排序元素的列表。

[Integer reply](../topics/protocol.md#resp-integers)：当指定 `store` 选项时，该命令返回 `destination` 列表中已排序元素的数量。