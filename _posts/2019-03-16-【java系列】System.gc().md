# System.gc()功能
说到System.gc()相信大家并不陌生，如果你调用了这个方法，那么JVM会尝试执行一次gc()。

多数情况下，我们并不需要使用这个方法，因为JVM的功能已经足够强大，它知道在什么时候触发gc是最合适的。但是有些场合下System.gc()还是有用的，比如下面的这个场景，
```
假设你的线上服务经常会在中午触发一次老年代gc, 而中午恰恰是你的业务高峰期，这种情况下会导致午高峰时期出现部分接口因为gc而超时，带来不好的影响。经过排查你发现在午高峰之前，老年代的容量就比较高了，经过午高峰恰恰达到了触发老年代gc的条件。
```
在上面的场景下你会怎么做？有的人会说做扩容，将服务的压力分散在更多的机器上。这么做的确能很大程度上缓解问题，但是无疑增大了研发的投入。还有没有别的方法呢？这时候System.gc()就派上用场了，我们可以在午高峰之前通过定时任务触发一次System.gc()，将老年代进行回收，那么午高峰就能较安全的度过了。

上面了说了这么多System.gc()的功能和场景，下面我们看一下System.gc()是怎么触发gc的，以及和它相关的一些JVM参数。

# 代码分析
```
    public static void gc() {
        Runtime.getRuntime().gc();
    }
```
System.gc()只是一个代理方法，实际上调用的是Runtime的gc()方法，Runtime.gc()是一个native方法，其实现在JVM中。下面我们看一下其实现，
```
JNIEXPORT void JNICALL
Java_java_lang_Runtime_gc(JNIEnv *env, jobject this)
{
    JVM_GC();
}

JVM_ENTRY_NO_ENV(void, JVM_GC(void))
  JVMWrapper("JVM_GC");
  if (!DisableExplicitGC) {
    Universe::heap()->collect(GCCause::_java_lang_system_gc);
  }
JVM_END
```
根据上面的代码可以看出，最终的清理过程是由Universe::heap()->collect()完成的，并且触发清理的理由是开发者显示的调用了System.gc()方法。由于Universe::heap()->collect()是通用的垃圾回收流程，这里不适合展开，所以我们只看一下针对_java_lang_system_gc有哪些特殊处理。

## 并行full gc
```
bool GenCollectedHeap::should_do_concurrent_full_gc(GCCause::Cause cause) {
  return UseConcMarkSweepGC &&
         ((cause == GCCause::_gc_locker && GCLockerInvokesConcurrent) ||
          (cause == GCCause::_java_lang_system_gc && ExplicitGCInvokesConcurrent));
}
```
当使用CMS gc并且开启了ExplicitGCInvokesConcurrent的时候，如果调用System.gc()，就会执行并行的full gc。
```
void GenCollectedHeap::collect_mostly_concurrent(GCCause::Cause cause) {
  assert(!Heap_lock->owned_by_self(), "Should not own Heap_lock");

  MutexLocker ml(Heap_lock);
  // Read the GC counts while holding the Heap_lock
  unsigned int full_gc_count_before = total_full_collections();
  unsigned int gc_count_before      = total_collections();
  {
    MutexUnlocker mu(Heap_lock);
    VM_GenCollectFullConcurrent op(gc_count_before, full_gc_count_before, cause);
    VMThread::execute(&op);
  }
}
```
那么什么是并行的full gc呢？这里有一篇文章[http://lovestblog.cn/blog/2015/05/07/system-gc/](http://lovestblog.cn/blog/2015/05/07/system-gc/)很好的解释了这个问题。

