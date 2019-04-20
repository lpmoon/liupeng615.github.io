多数人都知道当新生代对象经过多次垃圾回收的时候会进入到老年代，具体经过几次垃圾回收是垃圾回收器动态决定的。在hotspot中这个晋升阈值默认是15次，而这个值也会随着堆的具体占用情况而发生改变。比如我们经常可以在gc日志中看到这样的日志，
> Desired survivor size 536870912 bytes, new threshold 1 (max 15)  
> \- age   1: 1028367360 bytes, 1028367360 total
上面的日志告诉我们最新的晋升变成了1，也就是说下一次回收的时候已经经历过一次垃圾回收的对象都会进入到老年代。

# 源码分析
那么问题来了，这个值的变化规律是什么呢？下面看看hotspot jvm是如何实现的（下面的内容基于Openjdk 8），我们parNew算法为例，这是因为在java8中parNew和cms的新老代算法是比较经典的。阈值的调整位于parNewGeneration的collect方法中，当没有出现晋升失败的时候，会进入到调整阈值的代码，
```c++
if (!promotion_failed()) {
    // Swap the survivor spaces.
    eden()->clear(SpaceDecorator::Mangle);
    from()->clear(SpaceDecorator::Mangle);
    if (ZapUnusedHeapArea) {
      // This is now done here because of the piece-meal mangling which
      // can check for valid mangling at intermediate points in the
      // collection(s).  When a minor collection fails to collect
      // sufficient space resizing of the young generation can occur
      // an redistribute the spaces in the young generation.  Mangle
      // here so that unzapped regions don't get distributed to
      // other spaces.
      to()->mangle_unused_area();
    }
    swap_spaces();

    // A successful scavenge should restart the GC time limit count which is
    // for full GC's.
    size_policy->reset_gc_overhead_limit_count();

    assert(to()->is_empty(), "to space should be empty now");

    adjust_desired_tenuring_threshold();
  }
```
adjust_desired_tenuring_threshold()就是我们需要分析的函数，它的定义位于parNewGeneration的父类defNewGeneration中，
```c++
void DefNewGeneration::adjust_desired_tenuring_threshold() {
  // Set the desired survivor size to half the real survivor space
  _tenuring_threshold =
    age_table()->compute_tenuring_threshold(to()->capacity()/HeapWordSize);
}
```
最终的计算交给了ageTable，
```c++
uint ageTable::compute_tenuring_threshold(size_t survivor_capacity) {
  size_t desired_survivor_size = (size_t)((((double) survivor_capacity)*TargetSurvivorRatio)/100);
  size_t total = 0;
  uint age = 1;
  assert(sizes[0] == 0, "no objects with age zero should be recorded");
  while (age < table_size) {
    total += sizes[age];
    // check if including objects of age 'age' made us pass the desired
    // size, if so 'age' is the new threshold
    if (total > desired_survivor_size) break;
    age++;
  }
  uint result = age < MaxTenuringThreshold ? age : MaxTenuringThreshold;

  if (PrintTenuringDistribution || UsePerfData) {

    if (PrintTenuringDistribution) {
      gclog_or_tty->cr();
      gclog_or_tty->print_cr("Desired survivor size " SIZE_FORMAT " bytes, new threshold %u (max %u)",
        desired_survivor_size*oopSize, result, (int) MaxTenuringThreshold);
    }

    total = 0;
    age = 1;
    while (age < table_size) {
      total += sizes[age];
      if (sizes[age] > 0) {
        if (PrintTenuringDistribution) {
          gclog_or_tty->print_cr("- age %3u: " SIZE_FORMAT_W(10) " bytes, " SIZE_FORMAT_W(10) " total",
                                        age,    sizes[age]*oopSize,          total*oopSize);
        }
      }
      if (UsePerfData) {
        _perf_sizes[age]->set_value(sizes[age]*oopSize);
      }
      age++;
    }
    if (UsePerfData) {
      SharedHeap* sh = SharedHeap::heap();
      CollectorPolicy* policy = sh->collector_policy();
      GCPolicyCounters* gc_counters = policy->counters();
      gc_counters->tenuring_threshold()->set_value(result);
      gc_counters->desired_survivor_size()->set_value(
        desired_survivor_size*oopSize);
    }
  }

  return result;
}
```
ageTable的compute_tenuring_threshold方法干了下面的几件事情，
1. 首先计算desired_survivor_size，计算规则是survivor_capacity * TargetSurvivorRatio / 100，TargetSurvivorRatio默认情况下是50，也就是这个值默认情况下是survivor区域大小的一半。
2. 将ageTable中的各代的大小相加，直到相加的大小大于desired_survivor_size，并且记录此时的age大小
3. newTenuringThreshold = age < MaxTenuringThreshold > age : MaxTenuringThreshold

举个例子，
```
age为1的对象大小总和为1g
age为2的对象大小总和为1g
age为3的对象大小总和为1g

survivor大小为3g，size(age1) + size(age2) = 2g > 3g * 0.5，所以最新的晋升阈值为2。
```

从上面的内容可以看到新的晋升阈值和TargetSurvivorRatio有紧密的关系，
1. 如果TargetSurvivorRatio设置的很大比如100，也就是说当前survivor中只有最老的对象才会在下一次垃圾回收中晋升到老年代，这就有可能导致大量的老对象频繁的在两个survivor区域移动，造成资源的浪费，增加gc耗时。同时因为导致大量的老对象在survivor中，eden存活的对象在复制到survivor中的时候，会导致survivor空间不足。但是也会有相应的好处，那就是减少对象晋升到老年代的频率，充分利用survivor空间。
2. TargetSurvivorRatio设置的很小也不好，这是因为会导致survivor中对象很容晋升到老年代，造成老年代增速过快触发老年代gc，甚至full gc。同时造成空间的浪费，survivor只能使用TargetSurvivorRatio / 100 * size(survivor)

需要根据业务不同的场景决定是增大TargetSurvivorRatio还是调小TargetSurvivorRatio。一般情况下建议大家增大该值，提高资源（内存）的利用率。

# 举例
```java
package gc;

import java.util.ArrayList;
import java.util.List;

/**
 * Created by lpmoon on 2019/4/20.
 */
public class Threshold {
    static class LargeObject {
        private byte[] contents = new byte[1024 * 1024];
    }


    public static void main(String[] args) {
        List<LargeObject> objectList = new ArrayList<LargeObject>();
        for (int i = 0; i < 5000; i++) {
            objectList.add(new LargeObject());
        }
    }
}
```
上面的代码按照下面的参数运行后,
```
-Xms8g -Xmx8g -Xmn3g  -XX:SurvivorRatio=1 -XX:+PrintGCDetails  -XX:+UseConcMarkSweepGC -XX:+PrintTenuringDistribution 
```
可以看到如下的结果,
```
[GC (Allocation Failure) [ParNew
Desired survivor size 536870912 bytes, new threshold 1 (max 15)
- age   1: 1028367544 bytes, 1028367544 total
: 1048508K->1004305K(2097152K), 0.3998266 secs] 1048508K->1004305K(7340032K), 0.3998681 secs] [Times: user=0.51 sys=1.10, real=0.40 secs] 
[GC (Allocation Failure) [ParNew
Desired survivor size 536870912 bytes, new threshold 1 (max 15)
- age   1: 1072737104 bytes, 1072737104 total
: 2052746K->1048058K(2097152K), 0.7872071 secs] 2052746K->2052195K(7340032K), 0.7872333 secs] [Times: user=1.35 sys=3.24, real=0.78 secs] 
[GC (Allocation Failure) [ParNew
Desired survivor size 536870912 bytes, new threshold 1 (max 15)
- age   1: 1069596832 bytes, 1069596832 total
: 2096634K->1044619K(2097152K), 0.5284356 secs] 3100771K->3096348K(7340032K), 0.5284607 secs] [Times: user=1.46 sys=1.95, real=0.52 secs] 
[GC (Allocation Failure) [ParNew
Desired survivor size 536870912 bytes, new threshold 1 (max 15)
- age   1: 1062239904 bytes, 1062239904 total
: 2093195K->1037361K(2097152K), 0.5755425 secs] 4144924K->4133626K(7340032K), 0.5755721 secs] [Times: user=1.30 sys=1.69, real=0.57 secs] 
[GC (CMS Initial Mark) [1 CMS-initial-mark: 3096264K(5242880K)] 4134650K(7340032K), 0.0002949 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[CMS-concurrent-mark-start]
[CMS-concurrent-mark: 0.043/0.043 secs] [Times: user=0.08 sys=0.03, real=0.04 secs] 
[CMS-concurrent-preclean-start]
[CMS-concurrent-preclean: 0.008/0.008 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
[CMS-concurrent-abortable-preclean-start]
Heap
 par new generation   total 2097152K, used 2078323K [0x00000005c0000000, 0x0000000680000000, 0x0000000680000000)
  eden space 1048576K,  99% used [0x00000005c0000000, 0x00000005ff890688, 0x0000000600000000)
  from space 1048576K,  98% used [0x0000000600000000, 0x000000063f50c610, 0x0000000640000000)
  to   space 1048576K,   0% used [0x0000000640000000, 0x0000000640000000, 0x0000000680000000)
 concurrent mark-sweep generation total 5242880K, used 3096264K [0x0000000680000000, 0x00000007c0000000, 0x00000007c0000000)
 Metaspace       used 3317K, capacity 4500K, committed 4864K, reserved 1056768K
  class space    used 370K, capacity 388K, committed 512K, reserved 1048576K

Process finished with exit code 0
```
1. eden达到1g, 分配失败, 触发young gc
2. 复制对象到survivor0, survivor0使用率达到100%，新的晋升阈值变为1，下一次gc，这些对象都会进入到老年代
3. eden达到1g, 分配失败, 触发young gc。survivor0中的对象全部进入到老年代，同eden的对象复制到survivor1中，survivor1使用率达到100%，新的晋升阈值变为1，下一次gc，这些对象都会进入到老年代
4. 重复1.2.3

