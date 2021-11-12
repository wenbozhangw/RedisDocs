## EVALSHA sha1 numkeys [key [key ...]] [arg [arg ...]]

    起始版本：2.6.0。
    时间复杂度：取决于脚本执行。

根据给定的 SHA1 摘要，对缓存在服务器中的脚本进行求值。将脚本缓存到服务器的操作可以通过 [SCRIPT LOAD](SCRIPT LOAD.md) 命令进行。这个命令的其他地方，比如参数的传入方式，都和 [EVAL](EVAL.md) 命令一样。