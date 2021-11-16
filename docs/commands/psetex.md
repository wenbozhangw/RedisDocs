## PSETEX key milliseconds value

    起始版本：2.6.0
    时间复杂度：O(1)。

[PSETEX](psetex.md) 的工作方式与 [SETEX](setex.md) 完全一样，唯一的区别是过期时间以毫秒为单位而不是秒。

---

### Examples

```
redis> PSETEX mykey 1000 "Hello"
"OK"
redis> PTTL mykey
(integer) 1000
redis> GET mykey
"Hello"
redis> 
```