### CAS

CAS（Compare And Swap），即比较并替换，实现并发算法时常用到的一种技术。

#### CAS的思想

CAS的思想很简单：三个参数，一个当前内存值V，旧的预期值A，即将更新的值B。

当且仅当预期值A 和内存值V 相同时，将内存值修改为B，并返回TRUE，否则，什么都不做，并返回false。

当多个线程同时对某个资源进行CAS操作，只能有一个线程操作成功，但是并不会阻塞其他线程，其他线程只会收到操作失败的信号。可见 CAS 其实是一个乐观锁。

#### CAS是如何实现的

参考：https://segmentfault.com/a/1190000013127775

跟随AtomInteger的代码我们一路往下，就能发现最终调用的是 `sum.misc.Unsafe` 这个类。Unsafe是JDK底层的一个类，且限制了不允许你去实例化和使用它里边的方法。

如果用Unsafe.getUnsafe()来获取一个实例的话，在它的源码里会判断一下，如果当前是属于我们的应用系统，识别到有我们的那个类加载器后，就会报错，不让我们来获取实例。

再往下寻找我们会发现Unsafe的compareAndSwapInt是Native的方法：

```java
ublic final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

也就是说，这几个CAS的方法应该是使用了本地的方法。所以这几个方法的具体实现需要我们自己去jdk的源码中搜索。

下载一个 OpenJdk的源码，继续向下探索，我们发现在`/jdk9u/hotspot/src/share/vm/unsafe.cpp` 中有这样的代码：

```c
{CC "compareAndSetInt",   CC "(" OBJ "J""I""I"")Z",  FN_PTR(Unsafe_CompareAndSetInt)},
```

这个涉及到 JNI的调用，搜索 `Unsafe_CompareAndSetInt`后发现:

```c
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *)index_oop_from_field_offset_long(p, offset);

  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
} UNSAFE_END
```

最终我们终于看到了核心代码 `Atomic::cmpxchg`。

继续向底层探索，在文件`java/jdk9u/hotspot/src/os_cpu/linux_x86/vm/atomic_linux_x86.hpp`有这样的代码：

```c
inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value, cmpxchg_memory_order order) {
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```



我们通过文件名可以知道，针对不同的操作系统,JVM 对于 Atomic::cmpxchg 应该有不同的实现。由于我们服务基本都是使用的是64位linux，所以我们就看看linux_x86 的实现。



我们继续看代码：

- `__asm__` 的意思是这个是一段内嵌汇编代码。也就是在 C 语言中使用汇编代码。
- 这里的 `volatile`和 JAVA 有一点类似，但不是为了内存的可见性，而是告诉编译器对访问该变量的代码就不再进行优化。
- `LOCK_IF_MP(%4)` 的意思就比较简单，就是如果操作系统是多核的，那就增加一个 LOCK。
- `cmpxchgl` 就是汇编版的“比较并交换”。但是我们知道比较并交换，有三个步骤，不是原子的。所以在多核情况下加一个 LOCK，由CPU硬件保证他的原子性。
- 我们再看看 LOCK 是怎么实现的呢？我们去Intel的官网上看看，可以知道LOCK在的早期实现是直接将 cup 的总线阻塞，这样的实现可见效率是很低下的。后来优化为X86 cpu 有锁定一个特定内存地址的能力，当这个特定内存地址被锁定后，它就可以阻止其他的系统总线读取或修改这个内存地址。

关于 CAS 的底层探索我们就到此为止。总结一下 JAVA的 CAS是怎么实现的：

- java的 CAS 利用的是 底层Unsafe 这个类提供的 CAS 操作；
- Unsafe的 CAS 依赖的是 jvm 针对不同的操作系统实现的 Atomic::cmpxchg；
-  Atomic::cmpxchg 的实现使用了汇编的 CAS 操作，并使用 cpu 硬件提供的 lock 信号保证其原子性。

#### 原子操作

和CAS相关的一个概念是源自操作。原子操作是不可被中断的一个或一系列的操作。而CAS则是 Java 中保持原子操作的一种方式。

从Java1.5开始，JDK 的并发包里就提供了一些类来支持原子操作，都是以Atomic开头的。

volatile不能保证类似i++这样操作的原子性，CAS能保证。

##### 什么是原子操作

原子操作：一个或多个操作在CPU执行过程中不被中断的特性。

当我们说原子操作时，需要分清楚针对的是**CPU指令级别**还是**高级语言级别**。