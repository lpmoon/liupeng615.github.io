在阅读SoftReference的javadoc的时候可以看到如下的描述，
>  Soft reference objects, which are cleared at the discretion of the garbage
  collector in response to memory demand.  Soft references are most often used
  to implement memory-sensitive caches.

虚拟机会在有内存需求的时候回收SoftReference，那么究竟什么时候会回收呢？带着这个问题，来看一下JVM的源码。

GenCollectedHeap::do_collection是一个入口，
```
void GenCollectedHeap::do_collection(bool  full,
                                     bool   clear_all_soft_refs,
                                     size_t size,
                                     bool   is_tlab,
                                     int    max_level) {

    const bool do_clear_all_soft_refs = clear_all_soft_refs ||
                          collector_policy()->should_clear_all_soft_refs();

    for (int i = starting_level; i <= max_level; i++) {
        _gens[i]->collect(full, do_clear_all_soft_refs, size, is_tlab);
    }
}
```
如果传入的clear_all_soft_refs为true，则一定会进行SoftReference的回收。但是如果clear_all_soft_refs传入的false，也是有可能进行回收的，这取决于collector_policy()->should_clear_all_soft_refs()返回的是否为空。
```
  bool should_clear_all_soft_refs() { return _should_clear_all_soft_refs; }
  void set_should_clear_all_soft_refs(bool v) { _should_clear_all_soft_refs = v; 
```
那么这个set_should_clear_all_soft_refs这个方法又是什么时候调用的呢？
```
void AdaptiveSizePolicy::check_gc_overhead_limit(
                                          size_t young_live,
                                          size_t eden_live,
                                          size_t max_old_gen_size,
                                          size_t max_eden_size,
                                          bool   is_full_gc,
                                          GCCause::Cause gc_cause,
                                          CollectorPolicy* collector_policy) {

  // Ignore explicit GC's.  Exiting here does not set the flag and
  // does not reset the count.  Updating of the averages for system
  // GC's is still controlled by UseAdaptiveSizePolicyWithSystemGC.
  if (GCCause::is_user_requested_gc(gc_cause) ||
      GCCause::is_serviceability_requested_gc(gc_cause)) {
    return;
  }
  // eden_limit is the upper limit on the size of eden based on
  // the maximum size of the young generation and the sizes
  // of the survivor space.
  // The question being asked is whether the gc costs are high
  // and the space being recovered by a collection is low.
  // free_in_young_gen is the free space in the young generation
  // after a collection and promo_live is the free space in the old
  // generation after a collection.
  //
  // Use the minimum of the current value of the live in the
  // young gen or the average of the live in the young gen.
  // If the current value drops quickly, that should be taken
  // into account (i.e., don't trigger if the amount of free
  // space has suddenly jumped up).  If the current is much
  // higher than the average, use the average since it represents
  // the longer term behavor.
  const size_t live_in_eden =
    MIN2(eden_live, (size_t) avg_eden_live()->average());
  const size_t free_in_eden = max_eden_size > live_in_eden ?
    max_eden_size - live_in_eden : 0;
  const size_t free_in_old_gen = (size_t)(max_old_gen_size - avg_old_live()->average());
  const size_t total_free_limit = free_in_old_gen + free_in_eden;
  const size_t total_mem = max_old_gen_size + max_eden_size;
  const double mem_free_limit = total_mem * (GCHeapFreeLimit/100.0);
  const double mem_free_old_limit = max_old_gen_size * (GCHeapFreeLimit/100.0);
  const double mem_free_eden_limit = max_eden_size * (GCHeapFreeLimit/100.0);
  const double gc_cost_limit = GCTimeLimit/100.0;
  size_t promo_limit = (size_t)(max_old_gen_size - avg_old_live()->average());
  // But don't force a promo size below the current promo size. Otherwise,
  // the promo size will shrink for no good reason.
  promo_limit = MAX2(promo_limit, _promo_size);


  if (PrintAdaptiveSizePolicy && (Verbose ||
      (free_in_old_gen < (size_t) mem_free_old_limit &&
       free_in_eden < (size_t) mem_free_eden_limit))) {
    gclog_or_tty->print_cr(
          "PSAdaptiveSizePolicy::check_gc_overhead_limit:"
          " promo_limit: " SIZE_FORMAT
          " max_eden_size: " SIZE_FORMAT
          " total_free_limit: " SIZE_FORMAT
          " max_old_gen_size: " SIZE_FORMAT
          " max_eden_size: " SIZE_FORMAT
          " mem_free_limit: " SIZE_FORMAT,
          promo_limit, max_eden_size, total_free_limit,
          max_old_gen_size, max_eden_size,
          (size_t) mem_free_limit);
  }

  bool print_gc_overhead_limit_would_be_exceeded = false;
  if (is_full_gc) {
    if (gc_cost() > gc_cost_limit &&
      free_in_old_gen < (size_t) mem_free_old_limit &&
      free_in_eden < (size_t) mem_free_eden_limit) {
      // Collections, on average, are taking too much time, and
      //      gc_cost() > gc_cost_limit
      // we have too little space available after a full gc.
      //      total_free_limit < mem_free_limit
      // where
      //   total_free_limit is the free space available in
      //     both generations
      //   total_mem is the total space available for allocation
      //     in both generations (survivor spaces are not included
      //     just as they are not included in eden_limit).
      //   mem_free_limit is a fraction of total_mem judged to be an
      //     acceptable amount that is still unused.
      // The heap can ask for the value of this variable when deciding
      // whether to thrown an OutOfMemory error.
      // Note that the gc time limit test only works for the collections
      // of the young gen + tenured gen and not for collections of the
      // permanent gen.  That is because the calculation of the space
      // freed by the collection is the free space in the young gen +
      // tenured gen.
      // At this point the GC overhead limit is being exceeded.
      inc_gc_overhead_limit_count();
      if (UseGCOverheadLimit) {
        if (gc_overhead_limit_count() >=
            AdaptiveSizePolicyGCTimeLimitThreshold){
          // All conditions have been met for throwing an out-of-memory
          set_gc_overhead_limit_exceeded(true);
          // Avoid consecutive OOM due to the gc time limit by resetting
          // the counter.
          reset_gc_overhead_limit_count();
        } else {
          // The required consecutive collections which exceed the
          // GC time limit may or may not have been reached. We
          // are approaching that condition and so as not to
          // throw an out-of-memory before all SoftRef's have been
          // cleared, set _should_clear_all_soft_refs in CollectorPolicy.
          // The clearing will be done on the next GC.
          bool near_limit = gc_overhead_limit_near();
          if (near_limit) {
            collector_policy->set_should_clear_all_soft_refs(true);
            if (PrintGCDetails && Verbose) {
              gclog_or_tty->print_cr("  Nearing GC overhead limit, "
                "will be clearing all SoftReference");
            }
          }
        }
      }
      // Set this even when the overhead limit will not
      // cause an out-of-memory.  Diagnostic message indicating
      // that the overhead limit is being exceeded is sometimes
      // printed.
      print_gc_overhead_limit_would_be_exceeded = true;

    } else {
      // Did not exceed overhead limits
      reset_gc_overhead_limit_count();
    }
  }

  if (UseGCOverheadLimit && PrintGCDetails && Verbose) {
    if (gc_overhead_limit_exceeded()) {
      gclog_or_tty->print_cr("      GC is exceeding overhead limit "
        "of %d%%", GCTimeLimit);
      reset_gc_overhead_limit_count();
    } else if (print_gc_overhead_limit_would_be_exceeded) {
      assert(gc_overhead_limit_count() > 0, "Should not be printing");
      gclog_or_tty->print_cr("      GC would exceed overhead limit "
        "of %d%% %d consecutive time(s)",
        GCTimeLimit, gc_overhead_limit_count());
    }
  }
}
```
gc结束后会调用check_gc_overhead_limit方法检查gc的耗时是否超过某个阈值，如果is_full_gc并且查过了阈值，则会调用inc_gc_overhead_limit_count方法增加_gc_overhead_limit_count的值。如果_gc_overhead_limit_count的值超过了阈值AdaptiveSizePolicyGCTimeLimitThreshold-1的时候，则会设置需要回收SoftReference的标志位，下一次垃圾回收则会清理SoftReference。如果is_full_gc但是gc耗时没有超过阈值，则会重置_gc_overhead_limit_count。说到这里不少人可能有点懵，通俗点讲就是如果连续AdaptiveSizePolicyGCTimeLimitThreshold（默认为5）-1次gc的耗时超过了阈值，则下一次gc需要清理SoftReference。

```
  bool gc_overhead_limit_near() {
    return gc_overhead_limit_count() >=
        (AdaptiveSizePolicyGCTimeLimitThreshold - 1);
  }

  uint gc_overhead_limit_count() { return _gc_overhead_limit_count; }
  void reset_gc_overhead_limit_count() { _gc_overhead_limit_count = 0; }
  void inc_gc_overhead_limit_count() { _gc_overhead_limit_count++; 
```
上面说了collector_policy()->should_clear_all_soft_refs()为true的情况，那么do_collection什么时候传入clear_all_soft_refs为true呢？搜索代码可以发现，在satisfy_failed_allocation中有一处调用设置clear_all_soft_refs为true，此处代码的位置位于satisfy_failed_allocation最靠后的位置，此时jvm已经尝试过了包括gc、expand等多个手段，但是仍然分配出合适的空间，此时就需要尝试清理SoftReference来满足需求，所以此时传入的clear_all_soft_refs为true。

```

HeapWord* GenCollectorPolicy::satisfy_failed_allocation(size_t size,
                                                        bool   is_tlab) {
  GenCollectedHeap *gch = GenCollectedHeap::heap();
  GCCauseSetter x(gch, GCCause::_allocation_failure);
  HeapWord* result = NULL;

  assert(size != 0, "Precondition violated");
   
  省略其他代码

  // If we reach this point, we're really out of memory. Try every trick
  // we can to reclaim memory. Force collection of soft references. Force
  // a complete compaction of the heap. Any additional methods for finding
  // free memory should be here, especially if they are expensive. If this
  // attempt fails, an OOM exception will be thrown.
  {
    UIntFlagSetting flag_change(MarkSweepAlwaysCompactCount, 1); // Make sure the heap is fully compacted

    gch->do_collection(true             /* full */,
                       true             /* clear_all_soft_refs */,
                       size             /* size */,
                       is_tlab          /* is_tlab */,
                       number_of_generations() - 1 /* max_level */);
  }

  result = gch->attempt_allocation(size, is_tlab, false /* first_only */);
  if (result != NULL) {
    assert(gch->is_in_reserved(result), "result not in heap");
    return result;
  }

  assert(!should_clear_all_soft_refs(),
    "Flag should have been handled and cleared prior to this point");

  return NULL;
}
```

熟悉jvm代码的人知道处理do_collection，do_full_collection也是很多gc操作的入口。下面看一下什么情况下do_full_collection会清理SoftReference。
```
void VM_GenCollectFullConcurrent::doit() {
    gch->do_full_collection(gch->must_clear_all_soft_refs(), 0 /* collect only youngest gen */)
}

void VM_GenCollectFull::doit() {
    gch->do_full_collection(gch->must_clear_all_soft_refs(), _max_level);
}
```
上面两个任务分别是普通full gc和并发full gc的入口，他们都会调用must_clear_all_soft_refs检查是否需要清理SoftReference，

```
bool GenCollectedHeap::must_clear_all_soft_refs() {
  return _gc_cause == GCCause::_last_ditch_collection;
}
```
之前的文章提到过_last_ditch_collection多用于metaspace空间不足的情况。



总结下上面提到的集中情况，
1. 分配对象空间不足，jvm尝试了gc、expand多个操作后仍然无法满足需求
2. jvm基于统计信息，连续多次gc时间超过某个阈值，jvm认为下一次gc可能会超过时间（gc赶不上对象的分配速度），这时候需要清理SoftReference
3. metaspace空间不足