## 071、Java程序员的梦魇：线上系统突然挂掉，可怕的OOM内存溢出！

### Java程序员平时最常遇到的故障：系统OOM

考虑外部依赖的缓存、消息队列、数据库等东西挂掉，就我们自己系统本身而言，最常见挂掉的原因是什么？

其实就是系统OOM，也就是所谓的内存溢出！

**所谓的JVM OOM 内存溢出到底是什么？**

`其实说白了，也非常非常的简单，用一句话形容，你的JVM内存就这么点，结果你拼命的往里边塞东西，结果内存塞不下了，不就直接溢出了吗？`

看看下面这个图：

![JVM OOM内存溢出](071、Java程序员的梦魇：线上系统突然挂掉，可怕的OOM内存溢出！.assets/JVM OOM内存溢出.png)

一旦发生这个情况，就会导致你的系统直接停止运转，甚至会导致你的JVM进程直接崩溃掉，进程都没了！

这个时候对于线上看起来的场景就是，用户突然发现很奇怪，为什么点击APP，点击网页，都没反应了呢？