# 是什么
Oracle官网上对于AlwaysPreTouch的解释如下，
```
Pre-touch the Java heap during JVM initialization. Every page of the heap is thus demand-zeroed during initialization rather than incrementally during application execution.
```
当启用了该参数的时候，JVM启动阶段就会将堆初始化好，而不是在应用运行阶段动态增长。

# 应用场景
不少人会疑惑，这个参数有什么用处？如果在应用启动后，有大量的数据需要从新生代拷贝到老年代，并且你的应用属于对延迟敏感的应用，那么这个参数就会有用了。这是因为当大量的数据拷贝的时候，会涉及到大量的内存操作，包括内存申请。当启用了这个参数，这个阶段的内存申请就可以忽略了，直接分配好的内存做拷贝即可。

# 例子
```
public class GCTest {
    public static Map<Integer, LargeObject> holder = new HashMap();

    public static void main(String[] args) {
        for (int i = 0; i < 4000 * 1024; i++) {
            if (i % (1024 * 10) == 0) {
                try {
                    Thread.sleep(10);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            holder.put(i, new LargeObject());
        }
    }

    private static class LargeObject {
        byte[] content;
        public LargeObject() {
            content = new byte[1024];
        }
    }
}
```
使用如下的参数运行上面的程序
```
-Xmx5g
-Xms5g
-Xmn2g
-XX:PermSize=512m
-XX:MaxPermSize=512m
-XX:+UseConcMarkSweepGC
-XX:+UseParNewGC
-XX:SurvivorRatio=8
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-XX:+PrintGCCause
-XX:+PrintTenuringDistribution
-XX:+PrintSafepointStatistics
-XX:+PrintGCApplicationStoppedTime
```
可以得到如下的日志，
```
    2019-06-17T00:38:53.224-0800: Total time for which application threads were stopped: 0.0000963 seconds, Stopping threads took: 0.0000303 seconds
2019-06-17T00:38:54.228-0800: Total time for which application threads were stopped: 0.0001227 seconds, Stopping threads took: 0.0000299 seconds
2019-06-17T00:38:55.189-0800: [GC (Allocation Failure) 2019-06-17T00:38:55.189-0800: [ParNew
Desired survivor size 107347968 bytes, new threshold 1 (max 15)
- age   1:  214511352 bytes,  214511352 total
: 1677824K->209663K(1887488K), 1.6348205 secs] 1677824K->1553934K(5033216K), 1.6348809 secs] [Times: user=3.73 sys=2.10, real=1.64 secs] 
2019-06-17T00:38:56.824-0800: Total time for which application threads were stopped: 1.6350482 seconds, Stopping threads took: 0.0000188 seconds
2019-06-17T00:38:59.005-0800: [GC (Allocation Failure) 2019-06-17T00:38:59.005-0800: [ParNew
Desired survivor size 107347968 bytes, new threshold 1 (max 15)
- age   1:  214682016 bytes,  214682016 total
: 1887487K->209664K(1887488K), 0.9228701 secs] 3231758K->3284881K(5033216K), 0.9229299 secs] [Times: user=2.84 sys=2.94, real=0.92 secs] 
2019-06-17T00:38:59.928-0800: Total time for which application threads were stopped: 0.9230800 seconds, Stopping threads took: 0.0000358 seconds
```

两次young gc的时间分别是1.64s和0.92s。加上-XX:+AlwaysPreTouch再运行一次可以看到如下的日志，
```
2019-06-17T00:39:43.948-0800: Total time for which application threads were stopped: 0.0002106 seconds, Stopping threads took: 0.0000550 seconds
2019-06-17T00:39:45.174-0800: [GC (Allocation Failure) 2019-06-17T00:39:45.174-0800: [ParNew
Desired survivor size 107347968 bytes, new threshold 1 (max 15)
- age   1:  214511040 bytes,  214511040 total
: 1677824K->209664K(1887488K), 0.9713961 secs] 1677824K->1553935K(5033216K), 0.9714646 secs] [Times: user=3.72 sys=0.62, real=0.97 secs] 
2019-06-17T00:39:46.146-0800: Total time for which application threads were stopped: 0.9716097 seconds, Stopping threads took: 0.0000305 seconds
2019-06-17T00:39:47.070-0800: Total time for which application threads were stopped: 0.0002052 seconds, Stopping threads took: 0.0000513 seconds
2019-06-17T00:39:48.259-0800: [GC (Allocation Failure) 2019-06-17T00:39:48.259-0800: [ParNew
Desired survivor size 107347968 bytes, new threshold 1 (max 15)
- age   1:  214682752 bytes,  214682752 total
: 1887488K->209664K(1887488K), 0.4816425 secs] 3231759K->3284890K(5033216K), 0.4816978 secs] [Times: user=2.87 sys=0.02, real=0.49 secs] 
2019-06-17T00:39:48.741-0800: Total time for which application threads were stopped: 0.4818360 seconds, Stopping threads took: 0.0000286 seconds
```

可以看到两次gc的时间分别降到了0.97s和0.49s。

# OpenJDK中的实现
下面的代码基于OpenJDK8。

```
  if (pre_touch || AlwaysPreTouch) {
    os::pretouch_memory(previous_high, unaligned_new_high);
  }

```
如果启用了这个参数就会调用os::pretouch_memory进行预处理。
```
void os::pretouch_memory(char* start, char* end) {
  for (volatile char *p = start; p < end; p += os::vm_page_size()) {
    *p = 0;
  }
}
```
从start开始导致end，设置内容为0。

# 需要注意
启用这个参数会导致启动的时候变慢，尤其当堆设置的比较大的时候。