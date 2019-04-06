full gc的触发时机有哪些？这又是一个老生常谈的问题。。在google或者其他任意一个搜索引擎搜索，都会看到一大堆的文章，但是多数文章都是互相拷贝，或者凭着经验写出来的。为了彻底搞清楚这个问题，让我们来深入到代码一探究竟。PS:下面的代码基于Openjdk8。

首先我们需要明确的定义一下什么是full gc? ***我们认为同时触发了新生代gc和老年代gc，就叫full gc。***

full gc的一个入口位于GenCollectedHeap::collect(GCCause::Cause cause，
```
void GenCollectedHeap::collect(GCCause::Cause cause) {
  if (should_do_concurrent_full_gc(cause)) {
#if INCLUDE_ALL_GCS
    // mostly concurrent full collection
    collect_mostly_concurrent(cause);
#else  // INCLUDE_ALL_GCS
    ShouldNotReachHere();
#endif // INCLUDE_ALL_GCS
  } else if (cause == GCCause::_wb_young_gc) {
    // minor collection for WhiteBox API
    collect(cause, 0);
  } else {
#ifdef ASSERT
  if (cause == GCCause::_scavenge_alot) {
    // minor collection only
    collect(cause, 0);
  } else {
    // Stop-the-world full collection
    collect(cause, n_gens() - 1);
  }
#else
    // Stop-the-world full collection
    collect(cause, n_gens() - 1);
#endif
  }
}
```
该方法会启用一个异步线程执行VM_GenCollectFull，该任务会循环各代堆执行collect方法进行垃圾回收。

下面看一下哪些地方调用了这个方法，
![/assets/images/fullgc1.png](/assets/images/fullgc1.png)，
那么这些GCCause分别代表什么呢？

* _gc_locker  
  JNI进入临界区后，会阻塞gc的运行。当JNI退出临界区的时候，可能会触发一次gc操作，而这次gc的GCCause就是_gc_locker。
* _java_lang_system_gc  
  调用System.gc()
* _jvmti_force_gc  
  jvmti强制进行gc
* _last_ditch_collection
  如果java7(及以前)的perm或者java8的metaspace分配失败了，并且不能扩展的时候，则会触发_last_ditch_collection
* _wb_young_gc
  这个目前还不清楚是干啥的。。

full gc的另一个入口是CollectedHeap::collect_as_vm_thread(GCCause::Cause cause)，下面看一下哪些地方调用了这个方法，
![/assets/images/fullgc2.png](/assets/images/fullgc2.png)，这些gc的GCCause分别代表什么呢？

* _heap_inspection  
  获取堆统计信息的时候，执行gc
* _metadata_GC_threshold  
  当metadata分配失败，并且没有办法扩充的时候，会执行一次gc。gc的cause为_metadata_GC_threshold
* _last_ditch_collection  
  这个在上面已经提到过了。
* _heap_dump  
  如果用户执行了head dump，并且在dump的时候指定了-live参数，那么就会触发一次gc