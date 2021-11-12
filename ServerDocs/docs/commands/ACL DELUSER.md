## ACL DELUSER username [username ...]

    自 6.0.0 起可用。
    时间复杂度: O(1)考虑典型用户的平摊时间。

删除所有指定的 ACL 用户并终止所有与这些用户进行身份验证的连接。注意：特殊的默认用户不能从系统中删除，这是每个新连接都要使用的默认用户。用户列表可能包含不存在的用户名，在这种情况下，不对不存在的用户执行任何操作。

---

### Return Value

[Integer reply](../topics/protocol.md#RESP Integers) : 被删除的用户数。由于某些用户可能不存在，因此该数字并不总是与参数的数量相匹配。

---

### Examples

```
> ACL DELUSER antirez
1
```