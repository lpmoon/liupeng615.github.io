---
categories: Tomcat
tags: Tomcat
---

<!-- TOC -->

- [ThreadLocalLeakPreventionListener](#threadlocalleakpreventionlistener)
- [EngineConfig](#engineconfig)
- [HostConfig](#hostconfig)
- [JreMemoryLeakPreventionListener](#jrememoryleakpreventionlistener)
- [GlobalResourcesLifecycleListener](#globalresourceslifecyclelistener)
- [VersionLoggerListener](#versionloggerlistener)
- [SecurityListener](#securitylistener)
- [自定义Listener](#自定义listener)

<!-- /TOC -->
LifecycleListener用于定义生命周期事件监听器，监听的时间包括组件启动和停止等。LifecycleListener的定义如下，
```java
public interface LifecycleListener {


    /**
     * Acknowledge the occurrence of the specified event.
     *
     * @param event LifecycleEvent that has occurred
     */
    public void lifecycleEvent(LifecycleEvent event);


}
```
生命周期的各个阶段的定义位于Lifecycle中，各个阶段的状态转移如下所示，
```java
 *            start()
 *  -----------------------------
 *  |                           |
 *  | init()                    |
 * NEW -»-- INITIALIZING        |
 * | |           |              |     ------------------«-----------------------
 * | |           |auto          |     |                                        |
 * | |          \|/    start() \|/   \|/     auto          auto         stop() |
 * | |      INITIALIZED --»-- STARTING_PREP --»- STARTING --»- STARTED --»---  |
 * | |         |                                                            |  |
 * | |destroy()|                                                            |  |
 * | --»-----«--    ------------------------«--------------------------------  ^
 * |     |          |                                                          |
 * |     |         \|/          auto                 auto              start() |
 * |     |     STOPPING_PREP ----»---- STOPPING ------»----- STOPPED -----»-----
 * |    \|/                               ^                     |  ^
 * |     |               stop()           |                     |  |
 * |     |       --------------------------                     |  |
 * |     |       |                                              |  |
 * |     |       |    destroy()                       destroy() |  |
 * |     |    FAILED ----»------ DESTROYING ---«-----------------  |
 * |     |                        ^     |                          |
 * |     |     destroy()          |     |auto                      |
 * |     --------»-----------------    \|/                         |
 * |                                 DESTROYED                     |
 * |                                                               |
 * |                            stop()                             |
 * ----»-----------------------------»------------------------------
```
在大多数情况下tomcat中的组件出现生命周期迁移都会触发LifecycleBase的fireLifecycleEvent方法，该方法内部依次调用当前组件的各个listener的lifecycleEvent，
```java
    protected void fireLifecycleEvent(String type, Object data) {
        LifecycleEvent event = new LifecycleEvent(this, type, data);
        for (LifecycleListener listener : lifecycleListeners) {
            listener.lifecycleEvent(event);
        }
    }
```
Listener的定义需要在server.xml中，并且位于```<Server></Server>```标签内部，比如
```xml
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <!-- Security listener. Documentation at /docs/config/listeners.html
  <Listener className="org.apache.catalina.security.SecurityListener" />
  -->
  <!--APR library loader. Documentation at /docs/apr.html -->
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <!-- Prevent memory leaks due to use of particular java/javax APIs-->
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />
 
  。。。。。。
</Server>
```
定义的所有Listener都会出现在StandardServer的lifecycleListeners中，根据Listener定义的不同，有些Listener会传递到StandardServer的Child中去，比如ThreadLocalLeakPreventionListener会最终传递到Context中。

来看几个比较常用的Listener，

# ThreadLocalLeakPreventionListener
当Context停止的时候会调用到ThreadLocalLeakPreventionListener, 该Listener用于停止当前线程池中的所有线程，防止出现thread-local相关的内存泄漏。该Listener会调用stopIdleThreads方法，该方法内部会调用ThreadPoolExecutor的contextStopping方法停止所有线程。
```java
    public void contextStopping() {
        this.lastContextStoppedTime.set(System.currentTimeMillis());

        // save the current pool parameters to restore them later
        int savedCorePoolSize = this.getCorePoolSize();
        TaskQueue taskQueue =
                getQueue() instanceof TaskQueue ? (TaskQueue) getQueue() : null;
        if (taskQueue != null) {
            // note by slaurent : quite oddly threadPoolExecutor.setCorePoolSize
            // checks that queue.remainingCapacity()==0. I did not understand
            // why, but to get the intended effect of waking up idle threads, I
            // temporarily fake this condition.
            taskQueue.setForcedRemainingCapacity(Integer.valueOf(0));
        }

        // setCorePoolSize(0) wakes idle threads
        this.setCorePoolSize(0);

        // TaskQueue.take() takes care of timing out, so that we are sure that
        // all threads of the pool are renewed in a limited time, something like
        // (threadKeepAlive + longest request time)

        if (taskQueue != null) {
            // ok, restore the state of the queue and pool
            taskQueue.setForcedRemainingCapacity(null);
        }
        this.setCorePoolSize(savedCorePoolSize);
    }
```
上面代码首先将taskQueue的remainingCapacity调整为0，然后将corePoolSize调整为0，设置lastContextStoppedTime为当前时间。taskQueue为0，corePoolSize为0表示新提交的任务将由新创建的worker来完成。老的worker在通过获取task(getTask())的时候会进入到TaskQueue的take方法，该方法会判断该线程的创建时间是否早于laskContextStoppedTime，如果早于则终止该线程。

处理完之后还需要恢复现场，这样保证其他Context的请求能够得到正常的处理。

这样做之所有能够防止内存泄漏是因为ThreadLocal内部的资源是跟Thread绑定到一起的，强制停止所有线程后，跟ThreadLocal相关的资源就可以被回收了。

# EngineConfig

这个Listener一般由tomcat自动加载，不需要手工在server.xml中进行配置。该Listener位于StandardEngine中，目前只是用来打印相关启动日志

# HostConfig
这个Listener一般由tomcat自动加载，不需要手工在server.xml中进行配置。该Listener位于StandardHost中，主要用来加载相关app(context)，
```java
    /**
     * Deploy applications for any directories or WAR files that are found
     * in our "application root" directory.
     */
    protected void deployApps() {

        File appBase = host.getAppBaseFile();
        File configBase = host.getConfigBaseFile();
        String[] filteredAppPaths = filterAppPaths(appBase.list());
        // Deploy XML descriptors from configBase
        deployDescriptors(configBase, configBase.list());
        // Deploy WARs
        deployWARs(appBase, filteredAppPaths);
        // Deploy expanded folders
        deployDirectories(appBase, filteredAppPaths);

    }
```

# JreMemoryLeakPreventionListener

由于在默认情况context的类加载由WebappClassLoader完成。而WebappClassLoader类加载的默认优先级是子优先，也就是先由自己加载，这是为了保证各个context之间引用不同的版本jar时互相之间不受影响。但是在某些情况下会导致在context被销毁的时候，WebappClassLoader及其加载的类无法被回收。这个Listener就是用来解决这个问题，某些类的加载在Listener中会被代理给SystemClassLoader。

# GlobalResourcesLifecycleListener

用于创建全局的JNDI资源的Mbeans。

```java
    protected void createMBeans(String prefix, Context context)
        throws NamingException {

        if (log.isDebugEnabled()) {
            log.debug("Creating MBeans for Global JNDI Resources in Context '" +
                prefix + "'");
        }

        try {
            NamingEnumeration<Binding> bindings = context.listBindings("");
            while (bindings.hasMore()) {
                Binding binding = bindings.next();
                String name = prefix + binding.getName();
                Object value = context.lookup(binding.getName());
                if (log.isDebugEnabled()) {
                    log.debug("Checking resource " + name);
                }
                if (value instanceof Context) {
                    createMBeans(name + "/", (Context) value);
                } else if (value instanceof UserDatabase) {
                    try {
                        createMBeans(name, (UserDatabase) value);
                    } catch (Exception e) {
                        log.error("Exception creating UserDatabase MBeans for " + name,
                                e);
                    }
                }
            }
        } catch( RuntimeException ex) {
            log.error("RuntimeException " + ex);
        } catch( OperationNotSupportedException ex) {
            log.error("Operation not supported " + ex);
        }

    }
```
如果在server.xml中配置了对应的GlobalNamingResources，上面的代码中的bindings就不为空。

# VersionLoggerListener

用于打印tomcat、操作系统以及jvm相关参数。

# SecurityListener

权限相关校验，包括启动用户以及权限等。可以通过在server.xml的SecurityListener中配置checkedOsUsers来阻止某些用户启动tomcat

# 自定义Listener

```java
package org.apache.catalina.core;

import org.apache.catalina.*;
import org.apache.juli.logging.Log;
import org.apache.juli.logging.LogFactory;
import org.apache.tomcat.util.res.StringManager;


public class TestListener implements LifecycleListener,
        ContainerListener {

    private static final Log log =
        LogFactory.getLog(TestListener.class);

    private volatile boolean serverStopping = false;

    protected static final StringManager sm =
        StringManager.getManager(Constants.Package);

    @Override
    public void lifecycleEvent(LifecycleEvent event) {
        try {
            Lifecycle lifecycle = event.getLifecycle();
            if (Lifecycle.AFTER_START_EVENT.equals(event.getType()) &&
                    lifecycle instanceof Server) {
                // when the server starts, we register ourself as listener for
                // all context
                // as well as container event listener so that we know when new
                // Context are deployed
                Server server = (Server) lifecycle;
                registerListenersForServer(server);
            }

            if (Lifecycle.BEFORE_STOP_EVENT.equals(event.getType()) &&
                    lifecycle instanceof Server) {
                // Server is shutting down, so thread pools will be shut down so
                // there is no need to clean the threads
                serverStopping = true;
            }

            if (Lifecycle.AFTER_STOP_EVENT.equals(event.getType()) &&
                    lifecycle instanceof Context) {
                log.info("context " + ((Context) lifecycle).getName() + " stop");
            }
        } catch (Exception e) {
            String msg =
                sm.getString(
                    "TestListener.lifecycleEvent.error",
                    event);
            log.error(msg, e);
        }
    }

    @Override
    public void containerEvent(ContainerEvent event) {
        try {
            String type = event.getType();
            if (Container.ADD_CHILD_EVENT.equals(type)) {
                processContainerAddChild(event.getContainer(),
                    (Container) event.getData());
            } else if (Container.REMOVE_CHILD_EVENT.equals(type)) {
                processContainerRemoveChild(event.getContainer(),
                    (Container) event.getData());
            }
        } catch (Exception e) {
            String msg =
                sm.getString(
                    "TestListener.containerEvent.error",
                    event);
            log.error(msg, e);
        }

    }

    private void registerListenersForServer(Server server) {
        for (Service service : server.findServices()) {
            Engine engine = service.getContainer();
            engine.addContainerListener(this);
            registerListenersForEngine(engine);
        }

    }

    private void registerListenersForEngine(Engine engine) {
        for (Container hostContainer : engine.findChildren()) {
            Host host = (Host) hostContainer;
            host.addContainerListener(this);
            registerListenersForHost(host);
        }
    }

    private void registerListenersForHost(Host host) {
        for (Container contextContainer : host.findChildren()) {
            Context context = (Context) contextContainer;
            registerContextListener(context);
        }
    }

    private void registerContextListener(Context context) {
        context.addLifecycleListener(this);
    }

    protected void processContainerAddChild(Container parent, Container child) {
        if (log.isDebugEnabled())
            log.debug("Process addChild[parent=" + parent + ",child=" + child +
                "]");

        if (child instanceof Context) {
            registerContextListener((Context) child);
        } else if (child instanceof Engine) {
            registerListenersForEngine((Engine) child);
        } else if (child instanceof Host) {
            registerListenersForHost((Host) child);
        }

    }

    protected void processContainerRemoveChild(Container parent,
        Container child) {

        if (log.isDebugEnabled())
            log.debug("Process removeChild[parent=" + parent + ",child=" +
                child + "]");

        if (child instanceof Context) {
            Context context = (Context) child;
            context.removeLifecycleListener(this);
        } else if (child instanceof Host || child instanceof Engine) {
            child.removeContainerListener(this);
        }
    }
}

```

上面的TestListener会作用于所有的Context，当Context被删除的时候打印相关日志。在server.xml中配置如下，
```xml
 <Listener className="org.apache.catalina.core.TestListener" />
```
启动tomcat后删除一个context，日志会出现如下的数据，
```
信息: context /ROOT3 stop
```

