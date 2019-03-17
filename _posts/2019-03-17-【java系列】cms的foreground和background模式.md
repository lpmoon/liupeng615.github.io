在之前的一篇博客中提到了cms的foreground模式和background模式，那里只是一带而过，今天详细的介绍一下这二者的不同。

# foreground模式
通过调用System.gc()、分配对象内存不足等方式触发，当cms进入foreground模式的时候，整个流程是stop world的。foreground模式的入口是concurrentMarkSweepGeneration的collect方法，
```

void CMSCollector::collect(bool   full,
                           bool   clear_all_soft_refs,
                           size_t size,
                           bool   tlab)
{
  if (!UseCMSCollectionPassing && _collectorState > Idling) {
    // For debugging purposes skip the collection if the state
    // is not currently idle
    if (TraceCMSState) {
      gclog_or_tty->print_cr("Thread " INTPTR_FORMAT " skipped full:%d CMS state %d",
        Thread::current(), full, _collectorState);
    }
    return;
  }

  // The following "if" branch is present for defensive reasons.
  // In the current uses of this interface, it can be replaced with:
  // assert(!GC_locker.is_active(), "Can't be called otherwise");
  // But I am not placing that assert here to allow future
  // generality in invoking this interface.
  if (GC_locker::is_active()) {
    // A consistency test for GC_locker
    assert(GC_locker::needs_gc(), "Should have been set already");
    // Skip this foreground collection, instead
    // expanding the heap if necessary.
    // Need the free list locks for the call to free() in compute_new_size()
    compute_new_size();
    return;
  }
  acquire_control_and_collect(full, clear_all_soft_refs);
  _full_gcs_since_conc_gc++;
}
```
collect方法最终会调用acquire_control_and_collect来进行处理，下面我们看一下这个方法会干些什么，
```
void CMSCollector::acquire_control_and_collect(bool full,
        bool clear_all_soft_refs) {
  assert(SafepointSynchronize::is_at_safepoint(), "should be at safepoint");
  assert(!Thread::current()->is_ConcurrentGC_thread(),
         "shouldn't try to acquire control from self!");

  // Start the protocol for acquiring control of the
  // collection from the background collector (aka CMS thread).
  assert(ConcurrentMarkSweepThread::vm_thread_has_cms_token(),
         "VM thread should have CMS token");
  // Remember the possibly interrupted state of an ongoing
  // concurrent collection
  CollectorState first_state = _collectorState;

  // Signal to a possibly ongoing concurrent collection that
  // we want to do a foreground collection.
  _foregroundGCIsActive = true;

  // Disable incremental mode during a foreground collection.
  ICMSDisabler icms_disabler;

  // release locks and wait for a notify from the background collector
  // releasing the locks in only necessary for phases which
  // do yields to improve the granularity of the collection.
  assert_lock_strong(bitMapLock());
  // We need to lock the Free list lock for the space that we are
  // currently collecting.
  assert(haveFreelistLocks(), "Must be holding free list locks");
  bitMapLock()->unlock();
  releaseFreelistLocks();
  {
    MutexLockerEx x(CGC_lock, Mutex::_no_safepoint_check_flag);
    if (_foregroundGCShouldWait) {
      // We are going to be waiting for action for the CMS thread;
      // it had better not be gone (for instance at shutdown)!
      assert(ConcurrentMarkSweepThread::cmst() != NULL,
             "CMS thread must be running");
      // Wait here until the background collector gives us the go-ahead
      ConcurrentMarkSweepThread::clear_CMS_flag(
        ConcurrentMarkSweepThread::CMS_vm_has_token);  // release token
      // Get a possibly blocked CMS thread going:
      //   Note that we set _foregroundGCIsActive true above,
      //   without protection of the CGC_lock.
      CGC_lock->notify();
      assert(!ConcurrentMarkSweepThread::vm_thread_wants_cms_token(),
             "Possible deadlock");
      while (_foregroundGCShouldWait) {
        // wait for notification
        CGC_lock->wait(Mutex::_no_safepoint_check_flag);
        // Possibility of delay/starvation here, since CMS token does
        // not know to give priority to VM thread? Actually, i think
        // there wouldn't be any delay/starvation, but the proof of
        // that "fact" (?) appears non-trivial. XXX 20011219YSR
      }
      ConcurrentMarkSweepThread::set_CMS_flag(
        ConcurrentMarkSweepThread::CMS_vm_has_token);
    }
  }
  // The CMS_token is already held.  Get back the other locks.
  assert(ConcurrentMarkSweepThread::vm_thread_has_cms_token(),
         "VM thread should have CMS token");
  getFreelistLocks();
  bitMapLock()->lock_without_safepoint_check();
  if (TraceCMSState) {
    gclog_or_tty->print_cr("CMS foreground collector has asked for control "
      INTPTR_FORMAT " with first state %d", Thread::current(), first_state);
    gclog_or_tty->print_cr("    gets control with state %d", _collectorState);
  }

  // Check if we need to do a compaction, or if not, whether
  // we need to start the mark-sweep from scratch.
  bool should_compact    = false;
  bool should_start_over = false;
  decide_foreground_collection_type(clear_all_soft_refs,
    &should_compact, &should_start_over);

  if (first_state > Idling) {
    report_concurrent_mode_interruption();
  }

  set_did_compact(should_compact);
  if (should_compact) {
    // If the collection is being acquired from the background
    // collector, there may be references on the discovered
    // references lists that have NULL referents (being those
    // that were concurrently cleared by a mutator) or
    // that are no longer active (having been enqueued concurrently
    // by the mutator).
    // Scrub the list of those references because Mark-Sweep-Compact
    // code assumes referents are not NULL and that all discovered
    // Reference objects are active.
    ref_processor()->clean_up_discovered_references();

    if (first_state > Idling) {
      save_heap_summary();
    }

    do_compaction_work(clear_all_soft_refs);

    // Has the GC time limit been exceeded?
    DefNewGeneration* young_gen = _young_gen->as_DefNewGeneration();
    size_t max_eden_size = young_gen->max_capacity() -
                           young_gen->to()->capacity() -
                           young_gen->from()->capacity();
    GenCollectedHeap* gch = GenCollectedHeap::heap();
    GCCause::Cause gc_cause = gch->gc_cause();
    size_policy()->check_gc_overhead_limit(_young_gen->used(),
                                           young_gen->eden()->used(),
                                           _cmsGen->max_capacity(),
                                           max_eden_size,
                                           full,
                                           gc_cause,
                                           gch->collector_policy());
  } else {
    do_mark_sweep_work(clear_all_soft_refs, first_state,
      should_start_over);
  }
  // Reset the expansion cause, now that we just completed
  // a collection cycle.
  clear_expansion_cause();
  _foregroundGCIsActive = false;
  return;
}
```
上面的代码大概做了以下几件事情，
1. 设置_foregroundGCIsActive为true，表示foreground模式激活
2. 如果_foregroundGCShouldWait为true，则会调用        CGC_lock->wait(Mutex::_no_safepoint_check_flag);
进入阻塞状态状态，等着background模式执行完成后唤醒。
3. 调用decide_foreground_collection_type决定是否需要做压缩处理。当启动参数中设置了UseCMSCompactAtFullCollection，并且还需要满足以下其他几个条件中的任意一个，JVM才会判定这次gc需要做压缩处理。
   1. 上一次并发gc后的full gc次数大于CMSFullGCsBeforeCompaction
   2. 触发这次gc的原因是_java_lang_system_gc或者_jvmti_force_gc
   3. incremental_collection_will_fail返回true
```
 UseCMSCompactAtFullCollection &&
    ((_full_gcs_since_conc_gc >= CMSFullGCsBeforeCompaction) ||
     GCCause::is_user_requested_gc(gch->gc_cause()) ||
     gch->incremental_collection_will_fail(true /* consult_young */));
```
4. 如果需要做压缩的化，则调用do_compact_work做标记-清除-压缩操作
5. 否则调用do_mark_sweep_work做标记-清除操作。

# background模式
和foreground不一样，background模式是后台运行的，只有很少的环节需要stop world，比如initial mark和final remark阶段。除了运行模式的不同，其触发模式和background也不一样。触发由ConcurrentMarkSweepThread发起，该线程的主要部分由一个while的循环组成，
```
  while (!_should_terminate) {
    sleepBeforeNextCycle();
    if (_should_terminate) break;
    GCCause::Cause cause = _collector->_full_gc_requested ?
      _collector->_full_gc_cause : GCCause::_cms_concurrent_mark;
    _collector->collect_in_background(false, cause);
  }
```
循环内部首先调用sleepBeforeNextCycle()，该方法会调用_collector->shouldConcurrentCollect()判断是否需要进行gc。
```
bool CMSCollector::shouldConcurrentCollect() {
  if (_full_gc_requested) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print_cr("CMSCollector: collect because of explicit "
                             " gc request (or gc_locker)");
    }
    return true;
  }

  FreelistLocker x(this);

  // If the estimated time to complete a cms collection (cms_duration())
  // is less than the estimated time remaining until the cms generation
  // is full, start a collection.
  if (!UseCMSInitiatingOccupancyOnly) {
    if (stats().valid()) {
      if (stats().time_until_cms_start() == 0.0) {
        return true;
      }
    } else {
      // We want to conservatively collect somewhat early in order
      // to try and "bootstrap" our CMS/promotion statistics;
      // this branch will not fire after the first successful CMS
      // collection because the stats should then be valid.
      if (_cmsGen->occupancy() >= _bootstrap_occupancy) {
        if (Verbose && PrintGCDetails) {
          gclog_or_tty->print_cr(
            " CMSCollector: collect for bootstrapping statistics:"
            " occupancy = %f, boot occupancy = %f", _cmsGen->occupancy(),
            _bootstrap_occupancy);
        }
        return true;
      }
    }
  }

  // Otherwise, we start a collection cycle if
  // old gen want a collection cycle started. Each may use
  // an appropriate criterion for making this decision.
  // XXX We need to make sure that the gen expansion
  // criterion dovetails well with this. XXX NEED TO FIX THIS
  if (_cmsGen->should_concurrent_collect()) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print_cr("CMS old gen initiated");
    }
    return true;
  }

  // We start a collection if we believe an incremental collection may fail;
  // this is not likely to be productive in practice because it's probably too
  // late anyway.
  GenCollectedHeap* gch = GenCollectedHeap::heap();
  assert(gch->collector_policy()->is_two_generation_policy(),
         "You may want to check the correctness of the following");
  if (gch->incremental_collection_will_fail(true /* consult_young */)) {
    if (Verbose && PrintGCDetails) {
      gclog_or_tty->print("CMSCollector: collect because incremental collection will fail ");
    }
    return true;
  }

  if (MetaspaceGC::should_concurrent_collect()) {
      if (Verbose && PrintGCDetails) {
      gclog_or_tty->print("CMSCollector: collect for metadata allocation ");
      }
      return true;
    }

  return false;
}
```
上面的代码省略了一些无关紧要的部分，我们可以看到当满足下面的任意条件cms的background模式将被启动，
1. _full_gc_requested，当用户调用System.gc()并且设置了并发full gc模式，那么该参数就会被设置。
2. UseCMSInitiatingOccupancyOnly为false，并且统计数据有效，同时预估的完成一次cms gc的时间小于预估的老年代被填满的剩余时间
3. UseCMSInitiatingOccupancyOnly为false，并且统计数据无效，老年代的占有率大于_bootstrap_occupancy = ((double)CMSBootstrapOccupancy)/(double)100。CMSBootstrapOccupancy默认是50。
4. should_concurrent_collect()返回true
   1. 老年代的占有率大于某个阈值的时候，该阈值的计算如下所示
    ```
     _cmsGen ->init_initiating_occupancy(CMSInitiatingOccupancyFraction, CMSTriggerRatio);
    ```
    如果设置了CMSInitiatingOccupancyFraction，那么该值就是CMSInitiatingOccupancyFraction/100，否则就是((100 - MinHeapFreeRatio) + (double)(CMSTriggerRatio * MinHeapFreeRatio) / 100.0) / 100.0。默认情况下MinHeapFreeRatio = 40，CMSTriggerRatio = 80，所以阈值默认情况是0.92。
   2. expansion_cause() == CMSExpansionCause::_satisfy_allocation
   3. _cmsSpace->should_concurrent_collect()
5. incremental_collection_will_fail返回true
6. MetaspaceGC::should_concurrent_collect()返回true，这种情况下是metaspace分配失败的时候会触发这种场景。


当上面的任意条件满足的时候，JVM就会调用collect_in_background进行垃圾回收。后面的流程就是大家经常提到的初始化标记、并发标记、再标记等流程了。

# foreground和background
二者是互斥的，任何一个在执行的过程中，另一个都需等待对方执行完成。