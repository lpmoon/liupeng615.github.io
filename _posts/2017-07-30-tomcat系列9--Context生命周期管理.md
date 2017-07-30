tomcat管理着context的生命周期，如果在webapps目录下新增war包，则会启动新的context，如果删除war则会停止对应的context。这一切都是通过后台的一个check线程完成。这个线程在StandardEngine启动的时候初始化。
```
    @Override
    protected synchronized void startInternal() throws LifecycleException {

        // Log our server identification information
        if(log.isInfoEnabled())
            log.info( "Starting Servlet Engine: " + ServerInfo.getServerInfo());

        // Standard container startup
        super.startInternal();
    }
```
StardardEngine启动的时候会调用抽象类ContainerBase的startInternal方法，该方法的最后一步会调用threadStart尝试启动线程，
```
    protected void threadStart() {

        if (thread != null)
            return;
        if (backgroundProcessorDelay <= 0)
            return;

        threadDone = false;
        String threadName = "ContainerBackgroundProcessor[" + toString() + "]";
        thread = new Thread(new ContainerBackgroundProcessor(), threadName);
        thread.setDaemon(true);
        thread.start();

    }
```
这里说尝试启动线程是指该方法有可能不会启动线程，这取决于backgroundProcessorDelay这个值的大小。这个属性在StandardEngine被实例化的时候初始化成了10，10代表了每两次check之间间隔时间为10秒钟。ContainerBackgroundProcessor内部主要逻辑位于processChildren方法中，我们从方法的名字可以看出这个线程主要是针对子容器的处理。
```
        protected void processChildren(Container container) {
            ClassLoader originalClassLoader = null;

            try {
                if (container instanceof Context) {
					......
                }
                container.backgroundProcess();
                Container[] children = container.findChildren();
                for (int i = 0; i < children.length; i++) {
                    if (children[i].getBackgroundProcessorDelay() <= 0) {
                        processChildren(children[i]);
                    }
                }
            } catch (Throwable t) {
            }
        }
```
processChildren会遍历每个子容器，然后再递归调用processChildren处理这些子容器。StandardEngine的子容器是StandardHost，也就是说最终会进入到StandardHost的backgroundProcess。backgroundProcess方法位于容器的抽象类中，为容器提供后台处理能力，该方法最后会针对当前容器内部的各个LifecycleListener触发PERIODIC_EVENT这个事件。HostConfig是StandardHost所有LifecycleListener中的一个，也是真正管理context生命周期的LifecycleListener。HostConfig能够处理多种LifecycleEvent，PERIODIC_EVENT就是其中的一个。
```
        // Process the event that has occurred
        if (event.getType().equals(Lifecycle.PERIODIC_EVENT)) {
            check();
```
当HostConfig接收到PERIODIC_EVENT这个事件的时候，会调用check方法完成所有context的生命周期管理。
```
    /**
     * Check status of all webapps.
     */
    protected void check() {

        if (host.getAutoDeploy()) {
            // Check for resources modification to trigger redeployment
            DeployedApplication[] apps =
                deployed.values().toArray(new DeployedApplication[0]);
            for (int i = 0; i < apps.length; i++) {
                if (!isServiced(apps[i].name))
                    checkResources(apps[i], false);
            }

            // Check for old versions of applications that can now be undeployed
            if (host.getUndeployOldVersions()) {
                checkUndeploy();
            }

            // Hotdeploy applications
            deployApps();
        }
    }
```
check逻辑主要分为两个部分
1. 管理已经启动的context -> checkResources()
2. 进入热启动阶段 -> deployApps()

## 管理已经启动的context

首先看看check是如何处理已经启动的context的，
```
    protected synchronized void checkResources(DeployedApplication app,
            boolean skipFileModificationResolutionCheck) {
        String[] resources =
            app.redeployResources.keySet().toArray(new String[0]);
        // Offset the current time by the resolution of File.lastModified()
        long currentTimeWithResolutionOffset =
                System.currentTimeMillis() - FILE_MODIFICATION_RESOLUTION_MS;
        for (int i = 0; i < resources.length; i++) {
            long lastModified =
                    app.redeployResources.get(resources[i]).longValue();
            if (resource.exists() || lastModified == 0) {
                if (resource.lastModified() != lastModified && (!host.getAutoDeploy() ||
                        resource.lastModified() < currentTimeWithResolutionOffset ||
                        skipFileModificationResolutionCheck)) {
                    if (resource.isDirectory()) {
                        app.redeployResources.put(resources[i],
                                Long.valueOf(resource.lastModified()));
                    } else if (app.hasDescriptor &&
                            resource.getName().toLowerCase(
                                    Locale.ENGLISH).endsWith(".war")) {
                        Context context = (Context) host.findChild(app.name);
                        String docBase = context.getDocBase();
                        if (!docBase.toLowerCase(Locale.ENGLISH).endsWith(".war")) {
                            File docBaseFile = new File(docBase);
                            if (!docBaseFile.isAbsolute()) {
                                docBaseFile = new File(host.getAppBaseFile(),
                                        docBase);
                            }
                            reload(app, docBaseFile, resource.getAbsolutePath());
                        } else {
                            reload(app, null, null);
                        }
                        // Update times
                        app.redeployResources.put(resources[i],
                                Long.valueOf(resource.lastModified()));
                        app.timestamp = System.currentTimeMillis();
                        boolean unpackWAR = unpackWARs;
                        if (unpackWAR && context instanceof StandardContext) {
                            unpackWAR = ((StandardContext) context).getUnpackWAR();
                        }
                        if (unpackWAR) {
                            addWatchedResources(app, context.getDocBase(), context);
                        } else {
                            addWatchedResources(app, null, context);
                        }
                        return;
                    } else {
                        undeploy(app);
                        deleteRedeployResources(app, resources, i, false);
                        return;
                    }
                }
            } else {
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e1) {
                }
                if (resource.exists()) {
                    continue;
                }
                undeploy(app);
                deleteRedeployResources(app, resources, i, true);
                return;
            }
        }
        resources = app.reloadResources.keySet().toArray(new String[0]);
        boolean update = false;
        for (int i = 0; i < resources.length; i++) {
            File resource = new File(resources[i]);
            if (log.isDebugEnabled()) {
                log.debug("Checking context[" + app.name + "] reload resource " + resource);
            }
            long lastModified = app.reloadResources.get(resources[i]).longValue();
            if ((resource.lastModified() != lastModified &&
                    (!host.getAutoDeploy() ||
                            resource.lastModified() < currentTimeWithResolutionOffset ||
                            skipFileModificationResolutionCheck)) ||
                    update) {
                if (!update) {
                    // Reload application
                    reload(app, null, null);
                    update = true;
                }
                // Update times. More than one file may have been updated. We
                // don't want to trigger a series of reloads.
                app.reloadResources.put(resources[i],
                        Long.valueOf(resource.lastModified()));
            }
            app.timestamp = System.currentTimeMillis();
        }
    }
```
checkResources的代码比较长，不过大致可以分为两个阶段
1. redeployResources
2. reloadResources

### redeployResources

***获取context的redeployResources***，一般情况下会包含下面四个resource，
```
webapps\XXX.war
conf\Catalina\localhost\XXX.xml
webapps\ROOT
conf\context.xml
```
上面四个资源分别代表，
```
当前context的war
当前context配置
当前context解压后的内容
全局context配置
```

遍历context的redeployResources中的每个resource，分别判断resource是否还存在，

* 如果存在

1. 判断当前文件是否最近有过修改，如果有则进入步骤2，否则到6
2. 如果文件是文件夹则更新最近修改时间，否则进入步骤3
3. 如果在如果conf/catalina/localhost/中包含当前context的context.xml(如果context对应的是ROOT，则为ROOT.xml)，同时当前resource是以.war结尾的，也就是说war包被修改了，则进入步骤4，否则进入步骤5
4. 当前context的docBase是否以.war结尾，这个的意思是是说是否在启动的时候是否直接从.war加载context，而不解压.war。也就是说server.xml中是否配置了unpackWARs=”false“。如果是以.war结尾，则直接进行reload操作。否则先清除当前docBase目录，然后根据war重新加载Context。然后进入步骤6
5. 清除当前docBase目录，将当前context从host中移除。然后进入步骤6
6. 结束

* 如果不存在

1. 将当前context从host中移除。

下面是undeploy()函数的代码，根据context的name查找对应的context，调用context的stopInternal完成context的终止，然后将其从host的child中移除。

```
    private void undeploy(DeployedApplication app) {
        if (log.isInfoEnabled())
            log.info(sm.getString("hostConfig.undeploy", app.name));
        Container context = host.findChild(app.name);
        try {
            host.removeChild(context);
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            log.warn(sm.getString
                     ("hostConfig.context.remove", app.name), t);
        }
        deployed.remove(app.name);
    }
```

总的来说上面的步骤是针对被删除了或者修改过的context做移除或者reload操作。

### reloadResources
***获取context的reloadResources***，然后查看是否有修改，如果有则重新reaload资源。reloadResources定义在conf/context.xml以及当前context的自定义配置，这个自定义位于conf\Catalina\localhost\XXX.xml，XXX代表着context对应的名称。

比如全局的context.xml中定义着如下内容，
```
<Context>
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>
</Context>
```
那么系统则会监控WEB-INF/web.xml和/conf/web.xml这两个文件，一旦有改动则会尝试reload context。


处理完之后就进入到热启动阶段。

## 进入热启动阶段

热启动阶段会加载某些还没有加载的context。在正常情况下该步骤不会加载context，但是如果用户删除了某个context的docBase，并且该docBase对应的war还存在，这里就会重新解压对应的war，然后重新加载context

## 举例

下面举个例子来说明下上面的步骤，场景如下，
> webapps目录下有如下文件ROOT, ROOT.war

下面的所有操作都会触发刷新操作。

1. 删除ROOT

这会导致tomcat停止并且移除当前context，然后解压ROOT.war，重新加载context。打印日志如下，
```
信息: Undeploying context [/ROOT]
七月 30, 2017 8:17:59 下午 org.apache.catalina.core.TestListener lifecycleEvent
信息: context /ROOT stop
七月 30, 2017 8:17:59 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deploying web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT.war
七月 30, 2017 8:18:01 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deployment of web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT.war has finished in 2,298 ms
```

2. 删除ROOT.war

这会导致tomcat停止并且移除当前context, 同时删除ROOT目录。打印日志如下，
```
七月 30, 2017 8:19:01 下午 org.apache.catalina.startup.HostConfig undeploy
信息: Undeploying context [/ROOT]
七月 30, 2017 8:19:01 下午 org.apache.catalina.core.TestListener lifecycleEvent
信息: context /ROOT stop
```

3. 在ROOT.war中加入一个文件

如果conf/catalina/localhost/中不包含ROOT.xml，删除ROOT，重新解压war，同时重新加载context。打印日志如下，
```
七月 30, 2017 8:20:34 下午 org.apache.catalina.startup.HostConfig undeploy
信息: Undeploying context [/ROOT]
七月 30, 2017 8:20:34 下午 org.apache.catalina.core.TestListener lifecycleEvent
信息: context /ROOT stop
七月 30, 2017 8:20:34 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deploying web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT.war
七月 30, 2017 8:20:36 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deployment of web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT.war has finished in 2,168 ms
```
如果conf/catalina/localhost/中包含ROOT.xml，删除ROOT，重新解压war，同时重新加载context。日志如下，
```
七月 30, 2017 9:49:01 下午 org.apache.catalina.startup.HostConfig reload
信息: Reloading context [/ROOT]
七月 30, 2017 9:49:01 下午 org.apache.catalina.core.StandardContext reload
信息: Reloading Context with name [/ROOT] has started
七月 30, 2017 9:49:01 下午 org.apache.catalina.core.TestListener lifecycleEvent
信息: context /ROOT stop
七月 30, 2017 9:49:03 下午 org.apache.jasper.servlet.TldScanner scanJars
信息: At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
七月 30, 2017 9:49:03 下午 org.apache.catalina.core.StandardContext reload
信息: Reloading Context with name [/ROOT] is completed
```

4. 新增ROOT2.war

解压ROOT2.war为ROOT2，加载新的context。打印日志如下，
```
七月 30, 2017 8:21:46 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deploying web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT2.war
七月 30, 2017 8:21:48 下午 org.apache.jasper.servlet.TldScanner scanJars
信息: At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
七月 30, 2017 8:21:48 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deployment of web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT2.war has finished in 2,185 ms
```

5. 修改conf/context.xml

删除ROOT，重新解压缩war，同时重新加载context。打印日志如下，
```
七月 30, 2017 9:07:35 下午 org.apache.catalina.startup.HostConfig undeploy
信息: Undeploying context [/ROOT]
七月 30, 2017 9:07:35 下午 org.apache.catalina.core.TestListener lifecycleEvent
信息: context /ROOT stop
七月 30, 2017 9:07:35 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deploying web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT.war
七月 30, 2017 9:07:37 下午 org.apache.jasper.servlet.TldScanner scanJars
信息: At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
七月 30, 2017 9:07:37 下午 org.apache.catalina.startup.HostConfig deployWAR
信息: Deployment of web application archive D:\源码阅读\apache-tomcat-8.5.11-src\webapps\ROOT.war has finished in 2,274 ms
```

6. 修改conf/catalina/localhost/ROOT.xml

删除ROOT，重新解压缩war，同时重新加载context。打印日志如下，
```
七月 30, 2017 9:10:16 下午 org.apache.catalina.startup.HostConfig undeploy
信息: Undeploying context [/ROOT]
七月 30, 2017 9:10:16 下午 org.apache.catalina.core.TestListener lifecycleEvent
信息: context /ROOT stop
七月 30, 2017 9:10:16 下午 org.apache.catalina.startup.HostConfig deployDescriptor
信息: Deploying configuration descriptor D:\源码阅读\apache-tomcat-8.5.11-src\conf\Catalina\localhost\ROOT.xml
七月 30, 2017 9:10:19 下午 org.apache.jasper.servlet.TldScanner scanJars
信息: At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
七月 30, 2017 9:10:19 下午 org.apache.catalina.startup.HostConfig deployDescriptor
信息: Deployment of configuration descriptor D:\源码阅读\apache-tomcat-8.5.11-src\conf\Catalina\localhost\ROOT.xml has finished in 2,355 ms
```

7. 修改conf/web.xml

重新加载context。打印日志如下，
```
信息: Reloading context [/ROOT]
七月 30, 2017 9:41:21 下午 org.apache.catalina.core.StandardContext reload
信息: Reloading Context with name [/ROOT] has started
七月 30, 2017 9:41:21 下午 org.apache.catalina.core.TestListener lifecycleEvent
信息: context /ROOT stop
七月 30, 2017 9:41:24 下午 org.apache.jasper.servlet.TldScanner scanJars
信息: At least one JAR was scanned for TLDs yet contained no TLDs. Enable debug logging for this logger for a complete list of JARs that were scanned but no TLDs were found in them. Skipping unneeded JARs during scanning can improve startup time and JSP compilation time.
七月 30, 2017 9:41:24 下午 org.apache.catalina.core.StandardContext reload
信息: Reloading Context with name [/ROOT] is completed
```

