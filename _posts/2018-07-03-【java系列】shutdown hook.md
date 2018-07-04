# 什么是shutdown hook?
对于以java语言作为日常开发语言的开发者来说，ShutdownHook并不会显得陌生。用户在系统运行的时候，向虚拟机注册一些ShutdownHook，在虚拟机终止之前，虚拟机会调用所有的ShutdownHook，直到所有的ShutdownHook都完成后，才会终止虚拟机。

举个例子

```java
public class Hook {
    public static void main(String[] args) throws InterruptedException {
        Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("hook");
            }
        }));
    }
}
```
上面的代码在虚拟机终止之前会打印hook。

# shutdown hook的应用场景
上面说了ShutdownHook的作用，那么ShutdownHook有什么具体的应用场景呢？如果你拿这个问题去问面试者，很可能会把面试者问懵。虽然很多人都知道ShutdownHook，但是却很少使用。其实仔细想想，在日常的开发上线中，我们一直都在使用ShutdownHook。举个例子，
> 线上机器上线重启的时候，还有线上请求在处理，如果直接终止虚拟机，则会造成当前请求处理失败，更甚者导致某些数据的丢失。这时候ShutdownHook就能派上用场了：我们可以注册一个ShutdownHook，这个Hook用于关闭Socket或者设置一些标志位用于暂停接受新的请求，同时睡眠一定的时间，给虚拟机充足的时间处理完当前的请求。这样等该Hook结束的时候，虚拟机就可以正常关闭了--这就是我们通常所说的优雅关闭。

# shutdown hook原理
ShutdownHook在jdk层面的代码比较简单，我们通过addShutdownHook就可以注册我们需要的Hook,
```java
    public void addShutdownHook(Thread hook) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("shutdownHooks"));
        }
        ApplicationShutdownHooks.add(hook);
    }
```
addShutdownHook在做基本的权限校验后，调用ApplicationShutdownHooks.add()方法将Hook添加到一个map中,
```java
    static synchronized void add(Thread hook) {
        if(hooks == null)
            throw new IllegalStateException("Shutdown in progress");

        if (hook.isAlive())
            throw new IllegalArgumentException("Hook already running");

        if (hooks.containsKey(hook))
            throw new IllegalArgumentException("Hook previously registered");

        hooks.put(hook, hook);
    }
```
在虚拟机终止的时候会遍历这个map，然后调用所有Hook，runHooks方法会等待所有的hook线程结束，这是通过join操作完成的。
```java
    static void runHooks() {
        Collection<Thread> threads;
        synchronized(ApplicationShutdownHooks.class) {
            threads = hooks.keySet();
            hooks = null;
        }

        for (Thread hook : threads) {
            hook.start();
        }
        for (Thread hook : threads) {
            try {
                hook.join();
            } catch (InterruptedException x) { }
        }
    }
```
在ApplicationShutdownHooks初始化的时候会将runHooks方法注册到Shutdown的hooks中，
```java
    static {
        try {
            Shutdown.add(1 /* shutdown hook invocation order */,
                false /* not registered if shutdown in progress */,
                new Runnable() {
                    public void run() {
                        runHooks();
                    }
                }
            );
            hooks = new IdentityHashMap<>();
        } catch (IllegalStateException e) {
            // application shutdown hooks cannot be added if
            // shutdown is in progress.
            hooks = null;
        }
    }
```
Shutdown则是与虚拟机最直接交互的对象,
```java
   /* Invoked by the JNI DestroyJavaVM procedure when the last non-daemon
     * thread has finished.  Unlike the exit method, this method does not
     * actually halt the VM.
     */
    static void shutdown() {
        synchronized (lock) {
            switch (state) {
            case RUNNING:       /* Initiate shutdown */
                state = HOOKS;
                break;
            case HOOKS:         /* Stall and then return */
            case FINALIZERS:
                break;
            }
        }
        synchronized (Shutdown.class) {
            sequence();
        }
    }
```
根据注释我们可以看到在虚拟机调用DestroyJavaVM的时候，会调用shutdown方法，这个方法会继续调用sequence()完成Hook的调用。

下面来看一下jvm中DestroyJavaVM方法的实现，
```c++
jint JNICALL jni_DestroyJavaVM(JavaVM *vm) {
  jint res = JNI_ERR;
  DT_RETURN_MARK(DestroyJavaVM, jint, (const jint&)res);

  if (!vm_created) {
    res = JNI_ERR;
    return res;
  }

  JNIWrapper("DestroyJavaVM");
  JNIEnv *env;
  JavaVMAttachArgs destroyargs;
  destroyargs.version = CurrentVersion;
  destroyargs.name = (char *)"DestroyJavaVM";
  destroyargs.group = NULL;
  res = vm->AttachCurrentThread((void **)&env, (void *)&destroyargs);
  if (res != JNI_OK) {
    return res;
  }

  // Since this is not a JVM_ENTRY we have to set the thread state manually before entering.
  JavaThread* thread = JavaThread::current();
  ThreadStateTransition::transition_from_native(thread, _thread_in_vm);
  if (Threads::destroy_vm()) {
    // Should not change thread state, VM is gone
    vm_created = false;
    res = JNI_OK;
    return res;
  } else {
    ThreadStateTransition::transition_and_fence(thread, _thread_in_vm, _thread_in_native);
    res = JNI_ERR;
    return res;
  }
}
```
DestroyJavaVM会调用Threads::destroy_vm()完成虚拟机的销毁，
```c++
// Shutdown sequence:
//   + Shutdown native memory tracking if it is on
//   + Wait until we are the last non-daemon thread to execute
//     <-- every thing is still working at this moment -->
//   + Call java.lang.Shutdown.shutdown(), which will invoke Java level
//        shutdown hooks, run finalizers if finalization-on-exit
//   + Call before_exit(), prepare for VM exit
//      > run VM level shutdown hooks (they are registered through JVM_OnExit(),
//        currently the only user of this mechanism is File.deleteOnExit())
//      > stop flat profiler, StatSampler, watcher thread, CMS threads,
//        post thread end and vm death events to JVMTI,
//        stop signal thread
//   + Call JavaThread::exit(), it will:
//      > release JNI handle blocks, remove stack guard pages
//      > remove this thread from Threads list
//     <-- no more Java code from this thread after this point -->
//   + Stop VM thread, it will bring the remaining VM to a safepoint and stop
//     the compiler threads at safepoint
//     <-- do not use anything that could get blocked by Safepoint -->
//   + Disable tracing at JNI/JVM barriers
//   + Set _vm_exited flag for threads that are still running native code
//   + Delete this thread
//   + Call exit_globals()
//      > deletes tty
//      > deletes PerfMemory resources
//   + Return to caller
bool Threads::destroy_vm() {
  JavaThread* thread = JavaThread::current();

#ifdef ASSERT
  _vm_complete = false;
#endif
  // Wait until we are the last non-daemon thread to execute
  { MutexLocker nu(Threads_lock);
    while (Threads::number_of_non_daemon_threads() > 1 )
      Threads_lock->wait(!Mutex::_no_safepoint_check_flag, 0,
                         Mutex::_as_suspend_equivalent_flag);
  }

  // Hang forever on exit if we are reporting an error.
  if (ShowMessageBoxOnError && is_error_reported()) {
    os::infinite_sleep();
  }
  os::wait_for_keypress_at_exit();

  if (JDK_Version::is_jdk12x_version()) {
    // We are the last thread running, so check if finalizers should be run.
    // For 1.3 or later this is done in thread->invoke_shutdown_hooks()
    HandleMark rm(thread);
    Universe::run_finalizers_on_exit();
  } else {
    // run Java level shutdown hooks
    thread->invoke_shutdown_hooks();
  }

  before_exit(thread);

  thread->exit(true);

  // Stop VM thread.
  {
    MutexLocker ml(Heap_lock);

    VMThread::wait_for_vm_thread_exit();
    assert(SafepointSynchronize::is_at_safepoint(), "VM thread should exit at Safepoint");
    VMThread::destroy();
  }

  // clean up ideal graph printers
#if defined(COMPILER2) && !defined(PRODUCT)
  IdealGraphPrinter::clean_up();
#endif

#ifndef PRODUCT
  // disable function tracing at JNI/JVM barriers
  TraceJNICalls = false;
  TraceJVMCalls = false;
  TraceRuntimeCalls = false;
#endif

  VM_Exit::set_vm_exited();

  notify_vm_shutdown();

  delete thread;

  // exit_globals() will delete tty
  exit_globals();

  return true;
}
```
根据注释可以可以了解到destroy_vm完成了如下的工作
1. 等待其他的非守护进程完成
2. 调用java.lang.Shutdown.shutdown()，这是通过thread->invoke_shutdown_hooks()完成的
3. 等等

总结下大致的流程:
1. jdk注册Hook到Shutdown中
2. jvm终止的时候调用destroy_vm，destroy_vm调用Shutdown.shutdown()方法完成所有Hook的调用。

# 什么时候可以触发shutdown hook呢？
上面提到了ShutdownHook的调用时机是在虚拟机终止的时候，那么是任意情况都可以触发吗？下面总结一下什么场景可以触发，什么场景不可以触发。

## 可以触发
1. 程序正常终止
2. ctrl + c
3. Runtime.exit()
4. 使用kill pid的方式

## 不可以触发
1. kill -9 pid的方式
2. Runtime.halt()

什么场景下触发ShutdownHook和jvm对于信号的处理有关，这里暂时不展开，后面单独介绍一下这一部分。

# 使用shutdown hook需要注意哪些？

1. 关闭钩子本质上是一个线程（也称为Hook线程），对于一个JVM中注册的多个关闭钩子它们将会并发执行，所以JVM并不保证它们的执行顺序；由于是并发执行的，那么很可能因为代码不当导致出现竞态条件或死锁等问题，为了避免该问题，强烈建议在一个钩子中执行一系列操作。
2. Hook线程会延迟JVM的关闭时间，这就要求在编写钩子过程中必须要尽可能的减少Hook线程的执行时间，避免hook线程中出现耗时的计算、等待用户I/O等等操作。
3. 关闭钩子执行过程中可能被强制打断,比如在操作系统关机时，操作系统会等待进程停止，等待超时，进程仍未停止，操作系统会强制的杀死该进程，在这类情况下，关闭钩子在执行过程中被强制中止。
4. 在关闭钩子中，不能执行注册、移除钩子的操作，JVM将关闭钩子序列初始化完毕后，不允许再次添加或者移除已经存在的钩子，否则JVM抛出 IllegalStateException。
5. 不能在钩子调用System.exit()，否则卡住JVM的关闭过程，但是可以调用Runtime.halt()
6. Hook线程中同样会抛出异常，对于未捕捉的异常，线程的默认异常处理器处理该异常，不会影响其他hook线程以及JVM正常退出。