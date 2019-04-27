网上有很多介绍concurrent mode failure的文章，多数都是说当gc日志出现这个异常的时候表示新生代晋升失败。我相信很多人都有个疑问，为什么要叫这个名字（concurrent mode failure）。希望通过下面的内容能够给大家解惑。

想解答上面的问题最简单也是最直接的办法就是去阅读源码，首先我们在代码中搜索concurrent mode failure（下面的代码基于Openjdk8），可以看到相关日志的打印代码，
```
void CMSCollector::report_concurrent_mode_interruption() {
  if (is_external_interruption()) {
    if (PrintGCDetails) {
      gclog_or_tty->print(" (concurrent mode interrupted)");
    }
  } else {
    if (PrintGCDetails) {
      gclog_or_tty->print(" (concurrent mode failure)");
    }
    _gc_tracer_cm->report_concurrent_mode_failure();
  }
}
```
然后看一下哪里调用了这个方法，
```
void CMSCollector::acquire_control_and_collect(bool full, bool clear_all_soft_refs) {
    // 省略一大堆代码
    CollectorState first_state = _collectorState;
    if (first_state > Idling) {
       report_concurrent_mode_interruption();
    }
    // 省略一大堆代码
}
```
acquire_control_and_collect用于cms的foreground collect，foreground collect是stop world的，耗时较长。当进入该方法的时候，会检查_collectorState是否是处于Idling状态（Idling状态表示没有gc活动的初始状态），如果不是则会打印cocurrent mode failure的异常。正常情况下进入到acquire_control_and_collect时，_collectorState都应该是Idling，那么什么情况下不是呢？

当cms的background collect（concurrent mode）运行的时候，在不同的阶段会修改_collectorState的值，每一次修改状态后都会有一个检查是否需要退出background collect，
```
while (_collectorState != Idling) {
    if (TraceCMSState) {
      gclog_or_tty->print_cr("Thread " INTPTR_FORMAT " in CMS state %d",
        Thread::current(), _collectorState);
    }
    // The foreground collector
    //   holds the Heap_lock throughout its collection.
    //   holds the CMS token (but not the lock)
    //     except while it is waiting for the background collector to yield.
    //
    // The foreground collector should be blocked (not for long)
    //   if the background collector is about to start a phase
    //   executed with world stopped.  If the background
    //   collector has already started such a phase, the
    //   foreground collector is blocked waiting for the
    //   Heap_lock.  The stop-world phases (InitialMarking and FinalMarking)
    //   are executed in the VM thread.
    //
    // The locking order is
    //   PendingListLock (PLL)  -- if applicable (FinalMarking)
    //   Heap_lock  (both this & PLL locked in VM_CMS_Operation::prologue())
    //   CMS token  (claimed in
    //                stop_world_and_do() -->
    //                  safepoint_synchronize() -->
    //                    CMSThread::synchronize())

    {
      // Check if the FG collector wants us to yield.
      CMSTokenSync x(true); // is cms thread
      if (waitForForegroundGC()) {
        // We yielded to a foreground GC, nothing more to be
        // done this round.
        assert(_foregroundGCShouldWait == false, "We set it to false in "
               "waitForForegroundGC()");
        if (TraceCMSState) {
          gclog_or_tty->print_cr("CMS Thread " INTPTR_FORMAT
            " exiting collection CMS state %d",
            Thread::current(), _collectorState);
        }
        return;
      } else {
        // The background collector can run but check to see if the
        // foreground collector has done a collection while the
        // background collector was waiting to get the CGC_lock
        // above.  If yes, break so that _foregroundGCShouldWait
        // is cleared before returning.
        if (_collectorState == Idling) {
          break;
        }
      }
    }
}

bool CMSCollector::waitForForegroundGC() {
  bool res = false;
  assert(ConcurrentMarkSweepThread::cms_thread_has_cms_token(),
         "CMS thread should have CMS token");
  // Block the foreground collector until the
  // background collectors decides whether to
  // yield.
  MutexLockerEx x(CGC_lock, Mutex::_no_safepoint_check_flag);
  _foregroundGCShouldWait = true;
  if (_foregroundGCIsActive) {
    // The background collector yields to the
    // foreground collector and returns a value
    // indicating that it has yielded.  The foreground
    // collector can proceed.
    res = true;
    _foregroundGCShouldWait = false;
    ConcurrentMarkSweepThread::clear_CMS_flag(
      ConcurrentMarkSweepThread::CMS_cms_has_token);
    ConcurrentMarkSweepThread::set_CMS_flag(
      ConcurrentMarkSweepThread::CMS_cms_wants_token);
    // Get a possibly blocked foreground thread going
    CGC_lock->notify();
    if (TraceCMSState) {
      gclog_or_tty->print_cr("CMS Thread " INTPTR_FORMAT " waiting at CMS state %d",
        Thread::current(), _collectorState);
    }
    while (_foregroundGCIsActive) {
      CGC_lock->wait(Mutex::_no_safepoint_check_flag);
    }
    ConcurrentMarkSweepThread::set_CMS_flag(
      ConcurrentMarkSweepThread::CMS_cms_has_token);
    ConcurrentMarkSweepThread::clear_CMS_flag(
      ConcurrentMarkSweepThread::CMS_cms_wants_token);
  }
  if (TraceCMSState) {
    gclog_or_tty->print_cr("CMS Thread " INTPTR_FORMAT " continuing at CMS state %d",
      Thread::current(), _collectorState);
  }
  return res;
}

```

如果foreground collect模式启动了，则会设置_foregroundGCIsActive，同时检查_foregroundGCShouldWait是否为true，如果是则会进入睡眠状态等待唤醒。background collect在waitForForegroundGC方法中检查_foregroundGCIsActive是否为true，如果是则把_foregroundGCShouldWait设置为false，同时唤醒foreground collect模式，同时退出background collect。在这种情况下就出现了background collect(concurrent collect)没有执行完就退出的情况，而_collectorState可能不是Idling。

foreground collect启动后检测到_collectorState不是Idling，就知道上一个background collect(concurrent collect)没有执行完，所以打印出concurrent mode failure的error。

到这里相信大家已经能大概理解concurrent mode failure这个名字的由来了。那么什么情况下会出现两种模式的cms都同时运行呢？

假设在执行background collect的时候，出现了下面的两种情况:
1) 有大量的新生代提升到老年代，把老年代填满，后续新的晋升对象找不到合适的空间来存放
2) 有一个很大的对象直接分配到老年代，找不到合适的空间来存放

这两种情况都会触发foreground collect，而导致两种模式并发，就有可能触发concurrent mode failure的错误。