## ACL SETUSER username [rule [rule ...]]

    自 6.0.0 起可用。
    时间复杂度: 0(N)。其中 N 是提供的规则数。

创建具有指定规则的 ACL 用户或修改现有用户的规则。这是交互操作 Redis ACL 用户的主界面：如果用户名不存在，则该命令创建没有任何权限的用户名，然后从左到右读取作为连续参数提供的所有规则，将用户 ACL 规则设定为指定的。

如果用户已存在，则除了已设置的规则之外，还简单地应用提供的 ACL 规则。例如：

```
ACL SETUSER virginia on allkeys +set
```

上述命令将创建一个名为 virginia 的用户，该用户处于激活状态 (`on` 规则) ，可以访问任何 key (`allkeys` 规则)，并且可以调用 set 命令 (`+set` 规则)。然后另一个 SETUSER 调用可以修改用户规则：

```
ACL SETUSER virginia +get
```

上述规则不会将新规则覆盖用户 virginia，因此除了 [SET](set.md) 之外，用户 virginia 现在也可以使用 [GET](get.md) 命令。

当我们想确保从头开始定义一个用户，而不关心它之前是否有关联的规则时，我们可以使用特殊规则重置作为第一条规则，以便刷新所有其他现有规则：

```
ACL SETUSER antirez reset [... other rules ...]
```

重置用户后，返回到刚创建时的状态：non active(off rule)， 不能执行任何命令，不能访问任何key：

```
> ACL SETUSER antirez reset
+OK

>ACL LIST
1) "user antirez off -@all"
```

ACL 规则要么是诸如 "on"、"off"、"reset"、"allkeys"之类的词，要么是以特殊字符开头并后跟另一个字符串 (中间没有任何空格) 的特殊规则，例如 "+SET"。

以下文档是有关此命令的参考手册，但是我们的 [ACL tutorial](../topics/acl.md) 可能更温和地介绍了 ACL 系统的一般工作方式。

---

### List of rules

这是所有支持的 Redis ACL 规则的列表：

- on : 将用户设置为激活用户，可以使用 `AUTH <username> <password>` 验证此用户的身份。
- off : 将用户设置为非激活状态，将无法以该用户身份登录。请注意，如果用户在连接已通过此类用户进行身份验证后被禁用（设置为 off），则连接将继续按预期工作。要同时终止旧连接，您可以使用带有用户选项的 [CLIENT KILL](client-kill.md)。另一种方法是使用 [ACL DELUSER](acl-deluser.md) 删除用户，这将导致所有被认证为已删除用户的连接被断开。
- ~&lt;pattern&gt; : 将指定的 key pattern (glob style pattern，就像在 [KEYS](keys.md) 命令中一样)，添加到用户可访问的 key pattern 列表中。您可以向统一用户添加多个 key pattern。示例：~objects:*
- allkeys : ~* 的别名，它允许用户访问所有 key。
- resetkeys : 从用户可以访问的 key patterns 列表中删除所有 key patterns。
- &&lt;pattern&gt; : 将指定的 glob style pattern 添加到用户可访问的 Pub/Sub channel patterns 列表中。您可以向同一用户添加多个 channel patterns。示例: &chatroom:*
- allchannels : &* 的别名，它允许用户访问所有 Pub/Sub channels。
- resetchannels : 从用户可以访问的 Pub/Sub channel patterns 中删除所有 channel patterns。
- +&lt;command&gt; : 将此命令添加到用户可以调用的命令列表中。示例 : +zadd 。
- +@&lt;category&gt; : 将指定类别中的所有命令添加到用户能够执行的命令列表中。示例: +@string (添加所有字符串命令)。有关类别列表，请检查 [ACL CAT](acl-cat.md) 命令。
- +&lt;command&gt;|&lt;subcommand&gt; : 将指定的命令添加到用户可以执行的命令列表中，但仅限于指定的子命令。Example : +config|get 。如果指定用户的指定命令在完整版本中已经被允许，则生成错误。注意：没有删除子命令的对称命令，您需要删除整个命令并重新添加您要允许的子命令。这比删除子命令安全得多，未来 Redis 可能会添加新的危险子命令，因为减少配置不好。
- allcommands : +@all 的别名。添加服务器中存在的所有命令，包括由该用户执行的未来通过模块加载的命令。
- -&lt;command&gt; : 类似于 +&lt;command&gt; ，但删除命令而不是添加它。
- -@&lt;category&gt; : 类似于 +&lt;category&gt;，但删除类别中的所有命令而不是添加它们。
- nocommands : -@all 的别名。删除所有命令，用户将无法再执行任何操作。
- nopass : 用户被设置为 "no password" 用户。这意味着可以使用任何密码对此类用户进行身份验证。默认情况下，默认特殊用户设置为 "nopass"。nopass 规则还将为用户重置所有配置的密码。
- &gt;password : 在用户密码列表中添加指定的明文密码作为 hashed 密码。每个用户都可以有多个密码，这样密码轮换会更简单。指定的密码不会以明文形式存储在服务器中。示例：&gt;mypassword 。
- \#&lt;hashpassword&gt; : 将指定的 hash 密码添加到用户密码列表中。Redis hash 密码使用 SHA256 hash 并转换为十六进制字符串。示例：`#c3ab8ff13720e8ad9047dd39466b3c8974e592c2fa383d4a3960714caef0c4f2`。
- &lt;password: 类似 &gt;password 但删除密码而不是添加密码。
- !&lt;hashedpassword&gt; : 类似 \#&lt;hashpassword&gt; 但删除密码而不是添加密码。
- reset : 删除用户的任何能力。设置为 off，没有密码，无法执行任何命令，无法访问任何 key 。

---

### Return Value

[Simple string reply](../topics/protocol.md#RESP Simple Strings) : OK on success.

如果规则包含错误，则返回错误。

---

### History

- >= 6.2: Added Pub/Sub channel patterns.

---

### Examples

```
> ACL SETUSER alan allkeys +@string +@set -SADD >alanpassword
+OK

> ACL SETUSER antirez heeyyyy
(error) ERR Error in ACL SETUSER modifier 'heeyyyy': Syntax error
```