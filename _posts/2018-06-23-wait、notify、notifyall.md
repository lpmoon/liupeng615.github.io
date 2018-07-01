# wait、notify、notifyall介绍
wait、notify、notifyall是jdk提供的进程交互机制。当某个线程通过wait让出cpu后，其他线程可以通过notify将其唤醒。

而notifyall和notify的行为基本一致，不同的是notify唤醒一个线程而notifyall唤醒所有线程，这些线程重新开始竞争锁。

# wait、notify、notifyall一些需要注意的

1. 这三个函数在使用的时候都需要在synchronized关键字内部
2. notify执行后被唤醒的线程并不会立刻开始执行，而是会等到notify线程退出synchronized代码块后才开始重新执行，竞争锁
3. jdk的注释表明notify选择线程的过程是随机的，但是在官方的jdk版本中实际上并非随机的，而是按照wait的先后顺序唤醒，也就是说最先wait的线程最先被唤醒。jdk中的原话是 **The choice is arbitrary and occurs at the discretion of the implementation**

这里面的第三点往往是比较容易被人忽视的，如果你写的代码在唤醒的时候依赖随机性，那么很可能会在这里采坑。下面的这个例子可以比较好的解释这种现象，
```

public class Test2 {
    public static void main(String[] args) throws InterruptedException {
        Object o = new Object();

        for (int i = 0; i < 5; i++) {
            int finalI = i;
            new Thread(() -> {
                synchronized (o) {
                    String t = "t" + finalI;
                    System.out.println(t + " start");
                    try {
                        System.out.println(t + " wait");
                        o.wait();
                        System.out.println(t + " reRun");
                    } catch (InterruptedException e) {
                    }

                    o.notify();

                    try {
                        Thread.sleep((long) (Math.random() * 500));
                    } catch (InterruptedException e) {
                    }

                    System.out.println(t + " end");
                }
            }).start();

            Thread.sleep(500);
        }

        Thread.sleep(2000);

        new Thread(() -> {
            synchronized (o) {
                System.out.println("t start");
                o.notify();
                System.out.println("t notify");
                try {
                    System.out.println("t sleep");
                    Thread.sleep(2000);
                    System.out.println("t reRun");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("t end");
            }
        }).start();

    }
}

```

运行上面的代码可以看到线程的被唤醒顺序和其睡眠顺序是一致的。
```
t0 start
t0 wait
t1 start
t1 wait
t2 start
t2 wait
t3 start
t3 wait
t4 start
t4 wait
t start
t notify
t sleep
t reRun
t end
t0 reRun    <--  开始唤醒
t0 end
t1 reRun
t1 end
t2 reRun
t2 end
t3 reRun
t3 end
t4 reRun
t4 end
```

# wait、notify、notifyall原理

## wait

wait的实现位于JVM_MonitorWait，JVM_MonitorWait调用ObjectSynchronizer::wait()完成wait操作。


```c++
// -----------------------------------------------------------------------------
//  Wait/Notify/NotifyAll
// NOTE: must use heavy weight monitor to handle wait()
void ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {
  if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  if (millis < 0) {
    TEVENT (wait - throw IAX) ;
    THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
  }
  ObjectMonitor* monitor = ObjectSynchronizer::inflate(THREAD, obj());
  DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
  monitor->wait(millis, true, THREAD);

  /* This dummy call is in place to get around dtrace bug 6254741.  Once
     that's fixed we can uncomment the following line and remove the call */
  // DTRACE_MONITOR_PROBE(waited, monitor, obj(), THREAD);
  dtrace_waited_probe(monitor, obj, THREAD);
}
```

首先会将锁对象膨胀为重量锁monitor。这个操作是通过ObjectSynchronizer::inflate完成的，主要是将对象的mark修改ObjectMonitor。

在获取到重量锁后，调用锁的wait方法，重锁的wait方法比较长，下面摘取其中的关键几个步骤，
1. 通过CHECK_OWNER检查锁的持有者是否是当前线程，如果不是当前线程则会抛出IllegalMonitorStateException异常。这也就是为什么object.wait()的调用必须在synchronized中。
2. 检查线程是否已经被中断，如果已经被中断了则直接抛出异常。
3. 使用OrderAccess::fence()锁总线，保证不会重排。
4. 生成ObjectWaiter，然后将其添加到_WaitSet中，_WaitSet是一个双重链表。_WaitSet用于存放所有调用过wait方法的线程。
5. 使用ObjectMonitor::exit()释放锁。
6. 使用PlatformEvent::park()将当前线程挂起，等待其他线程调用notify唤醒当前线程。

## notify

 notify的实现位于JVM_MonitorNotify，JVM_MonitorNotify调用ObjectSynchronizer::notify()完成notify操作。

 ```
 void ObjectSynchronizer::notify(Handle obj, TRAPS) {
 if (UseBiasedLocking) {
    BiasedLocking::revoke_and_rebias(obj, false, THREAD);
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }

  markOop mark = obj->mark();
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    return;
  }
  ObjectSynchronizer::inflate(THREAD, obj())->notify(THREAD);
}

 ```

 首先会校验当前对象锁的持有线程是否是当前线程，如果不是则直接返回。然后调用将当前锁膨胀，膨胀过程和wait时一样。在所有前置功能完成后，调用ObjectMonitor::notify完成唤醒操作。

```c++
void ObjectMonitor::notify(TRAPS) {
  CHECK_OWNER();
  if (_WaitSet == NULL) {
     TEVENT (Empty-Notify) ;
     return ;
  }
  DTRACE_MONITOR_PROBE(notify, this, object(), THREAD);

  int Policy = Knob_MoveNotifyee ;

  Thread::SpinAcquire (&_WaitSetLock, "WaitSet - notify") ;
  ObjectWaiter * iterator = DequeueWaiter() ;
  if (iterator != NULL) {
     TEVENT (Notify1 - Transfer) ;
     guarantee (iterator->TState == ObjectWaiter::TS_WAIT, "invariant") ;
     guarantee (iterator->_notified == 0, "invariant") ;
     if (Policy != 4) {
        iterator->TState = ObjectWaiter::TS_ENTER ;
     }
     iterator->_notified = 1 ;
     Thread * Self = THREAD;
     iterator->_notifier_tid = Self->osthread()->thread_id();

     ObjectWaiter * List = _EntryList ;
     if (List != NULL) {
        assert (List->_prev == NULL, "invariant") ;
        assert (List->TState == ObjectWaiter::TS_ENTER, "invariant") ;
        assert (List != iterator, "invariant") ;
     }

     if (Policy == 0) {       // prepend to EntryList
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
             List->_prev = iterator ;
             iterator->_next = List ;
             iterator->_prev = NULL ;
             _EntryList = iterator ;
        }
     } else
     if (Policy == 1) {      // append to EntryList
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
            for (Tail = List ; Tail->_next != NULL ; Tail = Tail->_next) ;
            assert (Tail != NULL && Tail->_next == NULL, "invariant") ;
            Tail->_next = iterator ;
            iterator->_prev = Tail ;
            iterator->_next = NULL ;
        }
     } else
     if (Policy == 2) {      // prepend to cxq
         // prepend to cxq
         if (List == NULL) {
             iterator->_next = iterator->_prev = NULL ;
             _EntryList = iterator ;
         } else {
            iterator->TState = ObjectWaiter::TS_CXQ ;
            for (;;) {
                ObjectWaiter * Front = _cxq ;
                iterator->_next = Front ;
                if (Atomic::cmpxchg_ptr (iterator, &_cxq, Front) == Front) {
                    break ;
                }
            }
         }
     } else
     if (Policy == 3) {      // append to cxq
        iterator->TState = ObjectWaiter::TS_CXQ ;
        for (;;) {
            ObjectWaiter * Tail ;
            Tail = _cxq ;
            if (Tail == NULL) {
                iterator->_next = NULL ;
                if (Atomic::cmpxchg_ptr (iterator, &_cxq, NULL) == NULL) {
                   break ;
                }
            } else {
                while (Tail->_next != NULL) Tail = Tail->_next ;
                Tail->_next = iterator ;
                iterator->_prev = Tail ;
                iterator->_next = NULL ;
                break ;
            }
        }
     } else {
        ParkEvent * ev = iterator->_event ;
        iterator->TState = ObjectWaiter::TS_RUN ;
        OrderAccess::fence() ;
        ev->unpark() ;
     }

     if (Policy < 4) {
       iterator->wait_reenter_begin(this);
     }
  }

  Thread::SpinRelease (&_WaitSetLock) ;

  if (iterator != NULL && ObjectMonitor::_sync_Notifications != NULL) {
     ObjectMonitor::_sync_Notifications->inc() ;
  }
}
```

notify的操作大致可以分为以下的几步，
1. 通过CHECK_OWNER检查锁的持有者是否是当前线程，如果不是当前线程则会抛出IllegalMonitorStateException异常。这也就是为什么object.wait()的调用必须在synchronized中。 
2. 调用DequeueWaiter获取下一个需要唤醒的线程。
   ```
   inline ObjectWaiter* ObjectMonitor::DequeueWaiter() {
        // dequeue the very first waiter
        ObjectWaiter* waiter = _WaitSet;
        if (waiter) {
            DequeueSpecificWaiter(waiter);
        }
        return waiter;
    }
   ```
   waiter指向了链表_WaitSet的头，也就是第一个调用wait的第一个线程。
3. 根据不同的策略将获取到的第一个线程放入到_EntryList或者_cxq中。
4. 调用wait_reenter_begin

需要注意的是notify只是将进程添加到可运行队列中，而不是真的直接唤醒。实际的唤醒操作在ObjectMonitor::exit中，这个方法会在当前唤醒线程退出Synchronized代码块的时候调用。

实际的唤醒操作的代码再下面的函数中，
```
oid ObjectMonitor::ExitEpilog (Thread * Self, ObjectWaiter * Wakee) {
   assert (_owner == Self, "invariant") ;

   // Exit protocol:
   // 1. ST _succ = wakee
   // 2. membar #loadstore|#storestore;
   // 2. ST _owner = NULL
   // 3. unpark(wakee)

   _succ = Knob_SuccEnabled ? Wakee->_thread : NULL ;
   ParkEvent * Trigger = Wakee->_event ;

   // Hygiene -- once we've set _owner = NULL we can't safely dereference Wakee again.
   // The thread associated with Wakee may have grabbed the lock and "Wakee" may be
   // out-of-scope (non-extant).
   Wakee  = NULL ;

   // Drop the lock
   OrderAccess::release_store_ptr (&_owner, NULL) ;
   OrderAccess::fence() ;                               // ST _owner vs LD in unpark()

   if (SafepointSynchronize::do_call_back()) {
      TEVENT (unpark before SAFEPOINT) ;
   }

   DTRACE_MONITOR_PROBE(contended__exit, this, object(), Self);
   Trigger->unpark() ;

   // Maintain stats and report events to JVMTI
   if (ObjectMonitor::_sync_Parks != NULL) {
      ObjectMonitor::_sync_Parks->inc() ;
   }
}
```

## notifyall

notifyall的实现和notify基本类似，不同的是会顺序从_WaitSet中将线程取出，
```
  for (;;) {
     iterator = DequeueWaiter () ;
     if (iterator == NULL) break ;
     TEVENT (NotifyAll - Transfer1) ;
     ++Tally ;
     ......
  }
```

这些线程会再次进入锁竞争状态。

# 总结

上面的内容解释了notify/wait大致的工作原理，需要注意的是wait会将当前的对象锁膨胀为重量锁，在性能要求较为严格的场景下需要注意。