## SETNX key value

    起始版本：1.0.0。
    时间复杂度：O(1)。

将 key 的值设置为 value，如果 key 不存在，这种情况等同 [SET](set.md) 命令。当 key 存在，什么也不做。 [SETNX](setnx.md) 是 "**SET** if **N**ot e**X**ists" 的简写。

### Return Value

[Integer reply](../topics/protocol.md#resp-integers), 特定值：

- 1，如果 key 设置成功。
- 0，如果 key 设置失败。

### Examples

```
redis> SETNX mykey "Hello"
(integer) 1
redis> SETNX mykey "World"
(integer) 0
redis> GET mykey
"Hello"
redis> 
```

### Design pattern : Locking with SETNX

**Please note that:**

1. 不鼓励遵循以下模式，而是支持 [the Redlock algorithm](../topics/distlock.md)，该算法实施起来稍微复杂一点，但提供了更好的保证并且具有容错性。
2. 无论如何，我们保留旧的模式，因为某些现有实现链接到此页面作为参考。此外，这是一个有趣的示例，说明Redis命令能够被用来作为编程原语的。
3. 无论如何，即使假设一个单例的加锁原语，但是从 2.6.12 开始，可以创建一个更加简单的加锁原语，相当于使用 [SET](set.md) 命令来获取锁，并且用一个简单的 Lua 脚本来释放锁。该模式被记录在 [SET](set.md) 命令的页面中。

也就是说，[SETNX](setnx.md) 能够被使用并且以前也在被作为一个加锁原语使用。例如，获取 key 为 foo 的锁，客户端可以尝试以下操作：

```
SETNX lock.foo <current Unix time + lock timeout + 1>
```

如果客户端获得锁，[SETNX](setnx.md) 返回 1，那么 lock.foo key 的 Unix 时间设置为不在被认为有效的时间，客户端随后会使用 DEL lock.foo 去释放锁。

如果 [SETNX](setnx.md) 返回 0，那么该 key 已经被其他客户端锁定。如果这是一个非阻塞的锁，才能立即返回给调用者，或者尝试重新获取该锁，知道成功或者过期超时。

### Handling deadlocks

以上加锁算法存在一个问题：如果客户端出现故障，崩溃或者其他情况无法释放该锁会发生什么情况？由于 lock key 包含了 UNIX 时间戳，因此可以检测到这种情况。如果这样的时间戳等于当前的 Unix 时间，则锁不再有效。

当以下情况发生时，我们不能只根据 key 调用 [DEL](del.md) 来删除该锁，然后尝试执行 [SETNX](setnx.md)，因为这里存在竟态条件，当多个客户端察觉到一个过期的锁并且都尝试去释放它。

- C1 和 C2 读 lock.foo 检查时间戳，因为它们执行完 [SETNX](setnx.md) 后都被返回了 0，因为锁仍然被 C3 所持有，并且 C3 已经崩溃。
- C1 发送 DEL lock.foo
- C1 发送 SETNX lock.foo 命令并且成功返回
- C2 发送 DEL lock.foo
- C2 发送 SETNX lock.foo 命令并且成功返回
- **错误：**由于竟态条件导致 C1 和 C2 都获取到了锁。

幸运的是，可以使用以下的算法来避免这种情况，请看 C4 客户端所使用的更好的算法：

- C4 发送 SETNX lock.foo 为了获得锁
- 已经崩溃的客户端 C3 仍然持有该锁，所以 Redis 将会返回 0 给 C4
- C4 发送 GET lock.foo 检查该锁是否已经过期。如果没有过期，C4 客户端将会休眠一会儿，并且从头开始进行重试操作
- 另一种情况，如果因为 lock.foo key 的 Unix 时间小于当前的 Unix 时间而导致该锁已经过期，C4 会尝试执行以下的操作：
```
GETSET lock.foo <current Unix timestamp + lock timeout + 1>
```
- 由于 [GETSET](getset.md) 的语义，C4 会检查已经过期的旧值是否仍然存储在 lock.foo 中。如果是的话，C4 会获得锁。
- 如果另一个客户端，假如为 C5，比 C4 更快的通过 [GETSET](getset.md) 操作获取到锁，那么 C4 执行 [GETSET](getset.md) 操作会被返回一个不过期的时间戳。C4 将会从第一个步骤重新开始。请注意：即使 C4 在将来几秒设置该 key，这也不是问题。

为了使这种加锁算法更加健壮，持有锁的客户端应该总是要检查是否超时，保证使用 [DEL](del.md) 释放锁之前不会过期，因为客户端故障的情况可能是复杂的，不止是崩溃，还会阻塞一段时间，阻止一些操作的执行，并且在阻塞恢复后尝试执行 [DEL](del.md) （此时，该 LOCK 已经被其他客户端所持有）。