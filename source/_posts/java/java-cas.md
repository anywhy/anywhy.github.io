---
title: Java CAS 原理解析
categories: Java
tags:
  - Java
  - CAS
abbrlink: 485786759
date: 2019-10-16 12:32:29
---


## CAS操作原理分析

CAS:Compare and Swap, 翻译成比较并交换。
在Java中，`java.util.concurrent`包中借助了CAS实现了线程同步的一种乐观锁，区别于`synchronized`（悲观锁），不会导致需要锁的线程都挂起。

在Java中也提供了这种操作类，例如：

* AtomicBoolean
* AtomicInteger

<!-- more -->

具体如何使用呢？看个小小的demo，代码如下：

```java
public class Demo {
    static AtomicInteger ai = new AtomicInteger(0);

    public static void increment() {
        // 自增 1并返回之后的结果
        ai.incrementAndGet();
    }
}
```

### CAS 原理

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。CAS采用的是一种非阻塞算法（nonblocking algorithms），一个线程的失败或者挂起不应该影响其他线程的失败或挂起的算法。在Java中是通过调用JNI实现的。

例如上述demo中`AtomicInteger`实现:

```java
    public class AtomicInteger extends Number implements java.io.Serializable {
         ......
        // setup to use Unsafe.compareAndSwapInt for updates
        // 调用JNI的辅助类
        private static final Unsafe unsafe = Unsafe.getUnsafe();

         // 定义了volatile类型变量
         // 通过volatile关键字来保证多线程间数据的可见性的, 在没有锁的机制下可能需要借助volatile原语，保证线程间的数据是可见的（共享的）。这样才获取变量的值的时候才能直接读取。
         private volatile int value;

         .....
        public final boolean compareAndSet(int expect, int update) {
            // 利用JNI来完成CPU指令的操作
            return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
        }
    }
```

* 在上述实现用`unsafe.compareAndSwapInt`通过JNI调用了底层系统CUP指令实现原子性操作。
  * 代码片段如下：
  
   ```c
    // Adding a lock prefix to an instruction on MP machine
    // VC++ doesn't like the lock prefix to be on a single line
    // so we can't insert a label after the lock prefix.
    // By emitting a lock prefix, we can define a label after it.
    #define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                        __asm je L0      \
                        __asm _emit 0xF0 \
                        __asm L0:
    
    inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
    // alternative for InterlockedCompareExchange
    int mp = os::is_MP();
    __asm {
        mov edx, dest
        mov ecx, exchange_value
        mov eax, compare_value
        LOCK_IF_MP(mp)
        cmpxchg dword ptr [edx], ecx
    }
    }
  ``` 

  * `cmpxchg` 是汇编指令，用于比较并交换操作数。根据CPU处理器源代码所示，程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

## CAS可能出现的问题(ABA)

虽然这种 CAS 的机制能够保证increment() 方法，但依然有一些问题，例如，当线程A即将要执行第三步的时候，线程 B 把 i 的值加1，之后又马上把 i 的值减 1，然后，线程 A 执行第三步，这个时候线程 A 是认为并没有人修改过 i 的值，因为 i 的值并没有发生改变。而这，就是我们平常说的`ABA`问题。
对于基本类型的值来说，这种把数字改变了在改回原来的值是没有太大影响的，但如果是对于引用类型的话，就会产生很大的影响了。

如何解决ABA？
为了解决这个 ABA 的问题，我们可以引入版本控制，例如，每次有线程修改了引用的值，就会进行版本的更新，虽然两个线程持有相同的引用，但他们的版本不同，这样，我们就可以预防 `ABA` 问题了。Java 中提供了 AtomicStampedReference 这个类，就可以进行版本控制了。
