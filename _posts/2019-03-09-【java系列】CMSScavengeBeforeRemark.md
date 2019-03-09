# 背景
最近服务器出现了请求其他接口超时的情况，排查后发现是cms gc过长导致的，最长的CMS gc长达3s，需要定位是什么导致了CMS gc这么长，以及如何解决。

# 问题定位
根据cms的日志发现remark阶段耗时很长，因为remark阶段是
stop world的所以导致了其他接口超时。问题基本上定位了，下面需要看一下怎么优化了。根据日志可以看到在remark的时候young区基本上已经满了，而remark需要
根据young区作为gc root扫描老年代，所以当young区很大的时候会导致remark阶段超长。

# CMSScavengeBeforeRemark参数

## 是什么
在网上搜了下这种情况发现基本上都是推荐设置CMSScavengeBeforeRemark
参数，那么这个参数是干什么的呢？我们看一下官方的文档是怎么说的:
```
-XX:+CMSScavengeBeforeRemark
Enables scavenging attempts before the CMS remark step. By default, this option is disabled.
```
可以看到如果配置了这个参数，jvm会在remark之前尝试执行一次young gc，并且这个参数默认是关闭的。如果执行了young gc，那么对应的新生代的存活数量就会大大减少，remark需要的扫描时间也会减少。

## 验证
在线上配置了这个参数后，线上的CMS gc的remark阶段的max从3s降到了1s左右。

## 副作用
根据文档我们可以看到这个参数默认是关闭的，这是为什么呢？因为大部分应用young gc的增长频率并没有那么快，默认情况下开启这个参数反到会增加remark阶段的耗时。

## 原理
下面我们看一下jvm是如何处理这个参数的。首先找到remark阶段的函数,
```c++
void CMSCollector::checkpointRootsFinal(bool asynch,
  bool clear_all_soft_refs, bool init_mark_was_synchronous) {
  assert(_collectorState == FinalMarking, "incorrect state transition?");
  check_correct_thread_executing();
  // world is stopped at this checkpoint
  assert(SafepointSynchronize::is_at_safepoint(),
         "world should be stopped");
  TraceCMSMemoryManagerStats tms(_collectorState,GenCollectedHeap::heap()->gc_cause());

  verify_work_stacks_empty();
  verify_overflow_empty();

  SpecializationStats::clear();
  if (PrintGCDetails) {
    gclog_or_tty->print("[YG occupancy: "SIZE_FORMAT" K ("SIZE_FORMAT" K)]",
                        _young_gen->used() / K,
                        _young_gen->capacity() / K);
  }
  if (asynch) {
    if (CMSScavengeBeforeRemark) {
      GenCollectedHeap* gch = GenCollectedHeap::heap();
      // Temporarily set flag to false, GCH->do_collection will
      // expect it to be false and set to true
      FlagSetting fl(gch->_is_gc_active, false);
      NOT_PRODUCT(GCTraceTime t("Scavenge-Before-Remark",
        PrintGCDetails && Verbose, true, _gc_timer_cm, _gc_tracer_cm->gc_id());)
      int level = _cmsGen->level() - 1;
      if (level >= 0) {
        gch->do_collection(true,        // full (i.e. force, see below)
                           false,       // !clear_all_soft_refs
                           0,           // size
                           false,       // is_tlab
                           level        // max_level
                          );
      }
    }
    FreelistLocker x(this);
    MutexLockerEx y(bitMapLock(),
                    Mutex::_no_safepoint_check_flag);
    assert(!init_mark_was_synchronous, "but that's impossible!");
    checkpointRootsFinalWork(asynch, clear_all_soft_refs, false);
  } else {
    // already have all the locks
    checkpointRootsFinalWork(asynch, clear_all_soft_refs,
                             init_mark_was_synchronous);
  }
  verify_work_stacks_empty();
  verify_overflow_empty();
  SpecializationStats::print();
}
```
如果满足asynch，并CMSScavengeBeforeRemark开启的时候会进入到gch->do_collection方法。gch是分代回收的堆的抽象，其内部包含了young generation和old generation。同时由于_cmsGen->level()的返回值是1，所以level是0。
下面再来看一下do_collection的实现，这里只截取关键部分代码
```c++
    int starting_level = 0;
    if (full) {
      // Search for the oldest generation which will collect all younger
      // generations, and start collection loop there.
      for (int i = max_level; i >= 0; i--) { // 代码1
        if (_gens[i]->full_collects_younger_generations()) {
          starting_level = i;
          break;
        }
      }
    }

    bool must_restore_marks_for_biased_locking = false;

    int max_level_collected = starting_level;
    for (int i = starting_level; i <= max_level; i++) { // 代码2
      if (_gens[i]->should_collect(full, size, is_tlab)) {
        if (i == n_gens() - 1) {  // a major collection is to happen
          if (!complete) {
            // The full_collections increment was missed above.
            increment_total_full_collections();
          }
          pre_full_gc_dump(NULL);    // do any pre full gc dumps
        }
        ......
      }
      ......
    }
```
由于max_level传入的是0，所以代码1的循环只会执行一次，对应的_gens[0]为ParNewGeneration，full_collects_younger_generations的返回值是false，所以最终starting_level是0。也就是说代码2处的循环也是只会遍历一次，如果_gens[0]的should_collect方法返回的是true，则会往下走进行新生代的回收，因为full=true，所以should_collect返回的一定是true，也就是说一定会进入到ParNewGeneration的collect中。
```c++
 virtual bool should_collect(bool   full,
                              size_t word_size,
                              bool   is_tlab) {
    return (full || should_allocate(word_size, is_tlab));
  }
```

需要注意的是虽然设置了这个参数，JVM一定会进入到新生代的collect方法中，但是这不代表这一定会进行一次回收，这是因为collect方法内部还做了一次特殊的判断,
```
  // If the next generation is too full to accommodate worst-case promotion
  // from this generation, pass on collection; let the next generation
  // do it.
  if (!collection_attempt_is_safe()) {
    gch->set_incremental_collection_failed();  // slight lie, in that we did not even attempt one
    return;
  }
```
当老年代无法接收新生代晋升上去的对象的时候，这次young gc就会被放弃了。这也就是官方文档上强调**attempts**