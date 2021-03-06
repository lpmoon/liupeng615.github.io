---
categories: Java 
tags: Java 
---

在许多业务场景中都会有并发的对某一个数据进行自增的操作，比如统计请求数，在这种情况下就会面临着并发问题。

```

i = 0



线程1   线程2



i++     i++  



```

上面两个线程如果同时运行就会出现不可预测的结果，可能是1也可能是2，这都是因为每个线程都持有i的缓存并且由于i++在jvm指令中不是原子的，导致线程之间指令执行的时序性不确定。



在jdk1.5之前，我们为了保证数据的正确性往往需要使用synchronized关键字来保证同一时间只有一个线程能操作这个数据。而由于早期的synchronized没有偏向锁和轻量锁的概念，两个线程同一时间访问数据的时候，一个线程会进入睡眠状态而让出cpu，等待获取锁的线程的唤醒，在高并发情况下就会有大量的时间消耗在睡眠/唤醒的操作上。为了解决这一普遍性的问题，在jdk1.5之后引入AtomicXXX（AtomicInteger, AtomicLong等），根据命名我们很容易就能知道这些类都属于原子操作。并且AtomicXXX的性能要优于使用synchronized。为什么会出现这种情况呢，这里就需要研究下jdk是如何实现的了。以AtomicInteger为例，



```

    public final int incrementAndGet() {

        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;

    }

```

该函数会调用unsafe中的getAndAddInt操作，



```

    public final int getAndAddInt(Object var1, long var2, int var4) {

        int var5;

        do {

            var5 = this.getIntVolatile(var1, var2);

        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));



        return var5;

    }

```

从上面的代码可以看出jdk使用compareAndSwapInt操作保证每次更新的原子性。compareAndSwapInt这类操作可以统一用CAS这个元语来表示，

```

CAS newValue address oldValue

```

其表示的含义是如果传入的oldValue与address对应的内存数据相等，则设置address对应的数据为newValue，并且返回新的数据。如果不相等则返回老的数据。而整个CAS操作是原子性的



为了搞明白compareAndSwapInt是如何保证原子性的，我们需要再深入一点。由于该方法是native的，我们需要在jvm的源码中找出其实现。下面的jvm源码均是基于openjdk8-b132，在**hotspot/src/share/vm/prims**目录下找到unsafe.cpp文件，并且定位到下面的代码行，

```

UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))

  UnsafeWrapper("Unsafe_CompareAndSwapInt");

  oop p = JNIHandles::resolve(obj);

  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);

  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;

UNSAFE_END

```

可以看到该方法和新的内容均是由Atomic::cmpxchg完成的。由于不同的操作系统上支持的编译器不同，并且不同的硬件体系所支持的操作也不同，所以在jvm源码中有许多的cmpxchg实现。在***hotspot/src/os_cpu*目录下可以看到如下的目录结构，每个文件夹的前缀对应的是操作系统而后缀对应的是体系结构。在这些目录下都有对应的cmpxchg实现。



```

bsd_x86       bsd_zero      linux_sparc   linux_x86     linux_zero    solaris_sparc solaris_x86   windows_x86

```



比如linux_sparc中的实现是这样的 (代码来自atomic_linux_sparc.inline.hpp)



```

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {

  jint rv;

  __asm__ volatile(

    " cas    [%2], %3, %0"

    : "=r" (rv)

    : "0" (exchange_value), "r" (dest), "r" (compare_value)

    : "memory");

  return rv;

}

```

而linux_x86中的实现是这样的 (代码来自atomic_linux_x86.inline.hpp)



```

#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "



inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {

  int mp = os::is_MP();

  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"

                    : "=a" (exchange_value)

                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)

                    : "cc", "memory");

  return exchange_value;

}

```



由于linux_x86的组合我们比较常见，所以这里主要讲一下linux_x86里面的实现。



1. 判断当前系统是否是多处理器的，mp = os::is_MP()



2. 使用volatile保证后续的指令不会跟之前的指令进行重排而导致不可预测的结果。



3. 然后进入到函数体，这里需要先介绍下上面代码的整体结构

    ```

    LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"

    ```

    上面这行对应的实际的函数体。

    ```

     : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)

    ```

    上面这行对应的是函数体传入的参数，exchange_value作为参数1，存放在随机的一个寄存器内。compare_value作为参数2，存放在eax中。dest地址作为参数3，存放在随机的寄存器内。mp作为参数4，存放在随机的参数中。

    ```

    : "=a" (exchange_value)



    ```

    上面这行表示将eax对应的值写入到exchange_value中。



4. 先调用了LOCK_IF_MP宏。该宏的作用是如果当前系统是多处理器的则在后面的汇编之前加上lock;操作。



    > LOCK is not an instruction itself: it is an instruction prefix, which applies to the following instruction. That instruction must be something that does a read-modify-write on memory (INC, XCHG, CMPXCHG etc.) --- in this case it is the incl (%ecx) instruction which increments the long word at the address held in the ecx register.  

    The LOCK prefix ensures that the CPU has exclusive ownership of the appropriate cache line for the duration of the operation, and provides certain additional ordering guarantees. This may be achieved by asserting a bus lock, but the CPU will avoid this where possible. If the bus is locked then it is only for the duration of the locked instruction.



    通过上面的描述可以看到lock指令可以保证cpu在操作期间独享某些cache line并且保证了时序。其实总的来说就是保证了后面的操作可以看成一个原子操作。



4. 调用cmpxchgl操作完成对应的操作。该操作对应的语法是



    ```

    cmpxchg reg1, reg2/mem

    ```



    该指令的含义是使用eax和reg2/mem对应的数据进行比较，如果相同则将reg2/mem对应的数据设置为reg1里的值，否则设置eax对应的值为reg2/mem里面的值。



    对应到上面的代码%1对应的就是exchange_value(新值)，%3对应的dest(内存里的值)，而eax里面的值对应的compare_value（老值），这样如果compare_value和dest对应的值不相等，则把dest对应的值写入到eax中，否则把exchange_value的值写入到compar_value中。



    最终将eax的值写入到exchange_value中，也就是说compare成功则返回值是老的值，否则是dest中的值。





这样在Unsafe_CompareAndSwapInt函数中将cmpxchg的返回值和传入的compare_value进行对比，如果一样则表示成功了，否则就是失败。由上层拿到新的值重新发起CAS操作。





除了Unsafe_CompareAndSwapInt等使用汇编完成的多字节cas操作，jvm内部还提供了一个针对byte的cas操作，而该操作最终会被转化成多字节cas操作。



```

jbyte Atomic::cmpxchg(jbyte exchange_value, volatile jbyte* dest, jbyte compare_value) {

  assert(sizeof(jbyte) == 1, "assumption.");

  // 地址

  uintptr_t dest_addr = (uintptr_t)dest;

  // 计算起始地址在jint中的偏移量

  uintptr_t offset = dest_addr % sizeof(jint);

  // 对齐int地址 dest_int

  volatile jint* dest_int = (volatile jint*)(dest_addr - offset);

  // 获取值dest_int对应的int值cur

  jint cur = *dest_int;

  // 取地址为jbyte*

  jbyte* cur_as_bytes = (jbyte*)(&cur);

  // 一个新的值，该值等于cur

  jint new_val = cur;

  // 获取新值对应的地址

  jbyte* new_val_as_bytes = (jbyte*)(&new_val);

  // 修改新值中偏移为offset的byte为新的byte

  new_val_as_bytes[offset] = exchange_value;

  // 如果需要比较的byte不相同，则直接跳出循环

  while (cur_as_bytes[offset] == compare_value) {

    // 使用cmpxchg进行原数据的替换，这里的cmpxchg不是该函数。而是与操作系统有关的cmpxchg，

    // 该函数通过上面的ifdef引入，如果使用的windows系统，则该函数定义在os_windows.inline.hpp中

    jint res = cmpxchg(new_val, dest_int, cur);

    // 如果cmpxchg成功，则提跳出循环，否则修改新值得数据，进入下一次循环

    if (res == cur) break;

    cur = res;

    new_val = cur;

    new_val_as_bytes[offset] = exchange_value;

  }

  return cur_as_bytes[offset];

}

```
