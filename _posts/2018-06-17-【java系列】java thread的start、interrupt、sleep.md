# 线程启动

线程的启动分为两部分，一部分是jdk，另一部分是jvm。jdk部分负责基本的状态校验，实际的启动过程由jvm完成。

## jdk部分

线程启动在jdk的实现很简单，
```java
    public synchronized void start() {
        /**
         * This method is not invoked for the main method thread or "system"
         * group threads created/set up by the VM. Any new functionality added
         * to this method in the future may have to also be added to the VM.
         *
         * A zero status value corresponds to state "NEW".
         */
        if (threadStatus != 0)
            throw new IllegalThreadStateException();

        /* Notify the group that this thread is about to be started
         * so that it can be added to the group's list of threads
         * and the group's unstarted count can be decremented. */
        group.add(this);

        boolean started = false;
        try {
            start0();
            started = true;
        } finally {
            try {
                if (!started) {
                    group.threadStartFailed(this);
                }
            } catch (Throwable ignore) {
                /* do nothing. If start0 threw a Throwable then
                  it will be passed up the call stack */
            }
        }
    }
```

起初的时候会判断当前的线程的状态，如果不是NEW则抛出异常。然后将该线程加入到线程组中，之后调用native方法start0()完成线程的启动。

## jvm部分

```  
{"start0",   "()V",   (void *)&JVM_StartThread},
```

start0在jvm中对应的函数是JVM_StartThread，
```c++
JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  JavaThread *native_thread = NULL;

  // We cannot hold the Threads_lock when we throw an exception,
  // due to rank ordering issues. Example:  we might need to grab the
  // Heap_lock while we construct the exception.
  bool throw_illegal_thread_state = false;

  // We must release the Threads_lock before we can post a jvmti event
  // in Thread::start.
  {
    // Ensure that the C++ Thread and OSThread structures aren't freed before
    // we operate.
    MutexLocker mu(Threads_lock);

    // Since JDK 5 the java.lang.Thread threadStatus is used to prevent
    // re-starting an already started thread, so we should usually find
    // that the JavaThread is null. However for a JNI attached thread
    // there is a small window between the Thread object being created
    // (with its JavaThread set) and the update to its threadStatus, so we
    // have to check for this
    if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
    } else {
      // We could also check the stillborn flag to see if this thread was already stopped, but
      // for historical reasons we let the thread detect that itself when it starts running

      jlong size =
             java_lang_Thread::stackSize(JNIHandles::resolve_non_null(jthread));
      // Allocate the C++ Thread structure and create the native thread.  The
      // stack size retrieved from java is signed, but the constructor takes
      // size_t (an unsigned type), so avoid passing negative values which would
      // result in really large stacks.
      size_t sz = size > 0 ? (size_t) size : 0;
      native_thread = new JavaThread(&thread_entry, sz);

      // At this point it may be possible that no osthread was created for the
      // JavaThread due to lack of memory. Check for this situation and throw
      // an exception if necessary. Eventually we may want to change this so
      // that we only grab the lock if the thread was created successfully -
      // then we can also do this check and throw the exception in the
      // JavaThread constructor.
      if (native_thread->osthread() != NULL) {
        // Note: the current thread is not being used within "prepare".
        native_thread->prepare(jthread);
      }
    }
  }

  if (throw_illegal_thread_state) {
    THROW(vmSymbols::java_lang_IllegalThreadStateException());
  }

  assert(native_thread != NULL, "Starting null thread?");

  if (native_thread->osthread() == NULL) {
    // No one should hold a reference to the 'native_thread'.
    delete native_thread;
    if (JvmtiExport::should_post_resource_exhausted()) {
      JvmtiExport::post_resource_exhausted(
        JVMTI_RESOURCE_EXHAUSTED_OOM_ERROR | JVMTI_RESOURCE_EXHAUSTED_THREADS,
        "unable to create new native thread");
    }
    THROW_MSG(vmSymbols::java_lang_OutOfMemoryError(),
              "unable to create new native thread");
  }

  Thread::start(native_thread);

JVM_END
```

### 1. start防重入
```
MutexLocker mu(Threads_lock);
if (java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread)) != NULL) {
      throw_illegal_thread_state = true;
}
```
为了防止并发问题，首先引入一个mutex锁，这个锁是全局的，也就是说在一定程度上线程的启动是串行的。这个锁的作用范围到Thread::start(native_thread)之前。当锁对象的作用于结束后，会在对象的析构方法中自动释放锁。

如果当前线程成功获取了锁，会通过判断Thread的eetop这个字段来判断当前线程是否已经启动了。大家可能会想为什么不适用threadStatus来判断呢？上面代码的注释给出了原因:
> 由于jni attached的线程在对象创建和修改threadStatus之间有一个窗口，所以使用threadStataus来防重入不能完全满足要求。

threadStatus的更新是在Thread::start(native_thread)中完成的，不在前面提到的锁的作用范围内，所以使用threadStatus可能存在并发问题。
  
```
JavaThread* java_lang_Thread::thread(oop java_thread) {
  return (JavaThread*)java_thread->address_field(_eetop_offset);
}
``` 
 
### 2.设置stacksize  
从jdk中Thread的stackSize属性中获取栈大小，如果拿到的stacksize小于0则设置为0

### 3. 初始化JavaThread  

传入的任务入口为静态函数thread_entry。  
1. 设置thread_entry
2. 调用pthread_create创建线程，将tid存储在OSThread中，并且绑定JavaThread和OSThread之间的关系。线程的入口是java_start。
3. 当前线程会等待子线程完成初始化操作 
```c++
Monitor* sync_with_child = osthread->startThread_lock();
MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);
while ((state = osthread->get_state()) == ALLOCATED) {
    sync_with_child->wait(Mutex::_no_safepoint_check_flag);  
}  
```
只有等子线程的状态变为非ALLOCATED，父进程才会继续执行。

java_start会进行下面几个操作，  
1. 通过pthread_sigmask为当前线程屏蔽几个信号，包括SIGILL，SIGSEGV，SIGBUS等
2. 通知父进程已经初始化完成，唤醒父进程。然后等待父进程调用start_thread()
```
// handshaking with parent thread
  {
    MutexLockerEx ml(sync, Mutex::_no_safepoint_check_flag);

    // notify parent thread
    osthread->set_state(INITIALIZED);
    sync->notify_all();

    // wait until os::start_thread()
    while (osthread->get_state() == INITIALIZED) {
      sync->wait(Mutex::_no_safepoint_check_flag);
    }
  }
```
3. 等父进程调用start_thread()后，执行thread->run()，实际调用的是thread_entry。

### 4.开始执行任务
Thread::start(native_thread)
1. 更新threadStatus，该状态用于防止并发start
2. 唤起被阻塞的子线程
```
Monitor* sync_with_child = osthread->startThread_lock();  
MutexLockerEx ml(sync_with_child, Mutex::_no_safepoint_check_flag);  
sync_with_child->notify();  
```

到这里线程的启动流程基本上完成。

# 线程interrupt

在jvm中对应JVM_Interrupt，

```c++
JVM_ENTRY(void, JVM_Interrupt(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_Interrupt");

  // Ensure that the C++ Thread and OSThread structures aren't freed before we operate
  oop java_thread = JNIHandles::resolve_non_null(jthread);
  MutexLockerEx ml(thread->threadObj() == java_thread ? NULL : Threads_lock);
  // We need to re-resolve the java_thread, since a GC might have happened during the
  // acquire of the lock
  JavaThread* thr = java_lang_Thread::thread(JNIHandles::resolve_non_null(jthread));
  if (thr != NULL) {
    Thread::interrupt(thr);
  }
JVM_END
```
之后会调用os::interrupt，

```
void os::interrupt(Thread* thread) {
  assert(Thread::current() == thread || Threads_lock->owned_by_self(),
    "possibility of dangling Thread pointer");

  OSThread* osthread = thread->osthread();

  if (!osthread->interrupted()) {
    osthread->set_interrupted(true);
    // More than one thread can get here with the same value of osthread,
    // resulting in multiple notifications.  We do, however, want the store
    // to interrupted() to be visible to other threads before we execute unpark().
    OrderAccess::fence();
    ParkEvent * const slp = thread->_SleepEvent ;
    if (slp != NULL) slp->unpark() ;
  }

  // For JSR166. Unpark even if interrupt status already was set
  if (thread->is_Java_thread())
    ((JavaThread*)thread)->parker()->unpark();

  ParkEvent * ev = thread->_ParkEvent ;
  if (ev != NULL) ev->unpark() ;

}
```

可以看到如果当前线程没有被interrupt，则设置中断标志位。同时线程的_SleepEvent不为空，则调用_SleepEvent->unpark()中断线程。如果是java线程，则调用parker的unpark方法中断线程。

当调用Thread.Sleep()的时候线程通过_SleepEvent->park()挂起，
当调用Object.wait()的时候线程通过_ParkEvent->park()挂起。

# 线程Sleep

在jvm中对应JVM_Sleep，
```c++
JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
  JVMWrapper("JVM_Sleep");

  if (millis < 0) {
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }

  if (Thread::is_interrupted (THREAD, true) && !HAS_PENDING_EXCEPTION) {
    THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
  }

  // Save current thread state and restore it at the end of this block.
  // And set new thread state to SLEEPING.
  JavaThreadSleepState jtss(thread);

#ifndef USDT2
  HS_DTRACE_PROBE1(hotspot, thread__sleep__begin, millis);
#else /* USDT2 */
  HOTSPOT_THREAD_SLEEP_BEGIN(
                             millis);
#endif /* USDT2 */

  EventThreadSleep event;

  if (millis == 0) {
    // When ConvertSleepToYield is on, this matches the classic VM implementation of
    // JVM_Sleep. Critical for similar threading behaviour (Win32)
    // It appears that in certain GUI contexts, it may be beneficial to do a short sleep
    // for SOLARIS
    if (ConvertSleepToYield) {
      os::yield();
    } else {
      ThreadState old_state = thread->osthread()->get_state();
      thread->osthread()->set_state(SLEEPING);
      os::sleep(thread, MinSleepInterval, false);
      thread->osthread()->set_state(old_state);
    }
  } else {
    ThreadState old_state = thread->osthread()->get_state();
    thread->osthread()->set_state(SLEEPING);
    if (os::sleep(thread, millis, true) == OS_INTRPT) {
      // An asynchronous exception (e.g., ThreadDeathException) could have been thrown on
      // us while we were sleeping. We do not overwrite those.
      if (!HAS_PENDING_EXCEPTION) {
        if (event.should_commit()) {
          event.set_time(millis);
          event.commit();
        }
#ifndef USDT2
        HS_DTRACE_PROBE1(hotspot, thread__sleep__end,1);
#else /* USDT2 */
        HOTSPOT_THREAD_SLEEP_END(
                                 1);
#endif /* USDT2 */
        // TODO-FIXME: THROW_MSG returns which means we will not call set_state()
        // to properly restore the thread state.  That's likely wrong.
        THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
      }
    }
    thread->osthread()->set_state(old_state);
  }
  if (event.should_commit()) {
    event.set_time(millis);
    event.commit();
  }
#ifndef USDT2
  HS_DTRACE_PROBE1(hotspot, thread__sleep__end,0);
#else /* USDT2 */
  HOTSPOT_THREAD_SLEEP_END(
                           0);
#endif /* USDT2 */
JVM_END
```
方法开始进行了状态校验，如果线程已经被interrupted，则抛出异常。否则调用os::sleep()将当前线程变为睡眠状态。os::sleep()的本质是调用_SleepEvent的park方法将当前方法挂起。使用这种方式挂起的线程，可以通过interrupt()方法来中断该线程。

```java
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                } catch (InterruptedException e) {
                    System.out.println("interrupt");
                }

                System.out.println("hahaha");
            }
        });
        
        t1.start();

        t1.interrupt();
```
上面的代码可以输出interrupt和hahaha


