---
categories: Tomcat
tags: Tomcat
---

<!-- TOC -->

- [从缓存查找](#从缓存查找)
- [使用java的ExtClassLoader加载](#使用java的extclassloader加载)
- [如果有需要则代理给父类加载器](#如果有需要则代理给父类加载器)
- [搜索本地目录加载](#搜索本地目录加载)
- [代理给父类加载器](#代理给父类加载器)
- [总结](#总结)

<!-- /TOC -->

tomcat以Context作为基本单位部署应用，每个应用在tomcat中对应一个Context。为了使应用之间影响降到最低，每个应用都有自己的类加载器。java默认的类加载机制是父代理模式，也就是加载请求会一直往上代理给父加载器。但是tomcat默认的类加载器没有采用这种模式，下面来看一下tomcat默认的类加载器WebappClassLoader是如何实现的。

整个过程可以大致分为下面的几个步骤，
1. 从缓存查找
2. 使用java的ExtClassLoader加载
3. 如果有需要则代理给父类加载器
4. 搜索本地目录加载
5. 代理给父类加载器

下面分别解释下上面的步骤做了什么，
# 从缓存查找

从缓存查找主要分为两步，一步是从当前的类加载器的内部缓存查找，另一步是从虚拟机的缓存中查找。
```java
            // (0) Check our previously loaded local class cache
            clazz = findLoadedClass0(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Returning class from cache");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }

            // (0.1) Check our previously loaded class cache
            clazz = findLoadedClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Returning class from cache");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
```
WebAppClassLoader重写了findLoadedClass0方法，这样做好处是首先从类加载器的内部缓存查找对应的类，
```java
    protected Class<?> findLoadedClass0(String name) {

        String path = binaryNameToPath(name, true);

        ResourceEntry entry = resourceEntries.get(path);
        if (entry != null) {
            return entry.loadedClass;
        }
        return null;
    }
```
上面代码的resourceEntries就是类加载器的内部缓存，记录了类路径对应的资源以及加载的类，这个缓存会在步骤4中得到更新。如果本地缓存没有，则调动父类的findLoadedClass查找类，这个方法会调用jvm提供的native方法，进行类查找。

**之所以会有两层缓存，我认为是存在并发的情况。在某些场景下步骤4加载完缓存后还没来得及更新本地缓存，这时候另一个类加载请求在本地缓存中查找不到类，如果没有jvm层面的这次缓存查找，则会进入到复杂的类加载环节中，降低整体的性能。**

# 使用java的ExtClassLoader加载

这一步是为了防止WebappClassLoader覆盖Java SE的类。
```java
 ClassLoader javaseLoader = getJavaseClassLoader();
            boolean tryLoadingFromJavaseLoader;
            try {
                // Use getResource as it won't trigger an expensive
                // ClassNotFoundException if the resource is not available from
                // the Java SE class loader. However (see
                // https://bz.apache.org/bugzilla/show_bug.cgi?id=58125 for
                // details) when running under a security manager in rare cases
                // this call may trigger a ClassCircularityError.
                tryLoadingFromJavaseLoader = (javaseLoader.getResource(resourceName) != null);
            } catch (ClassCircularityError cce) {
                // The getResource() trick won't work for this class. We have to
                // try loading it directly and accept that we might get a
                // ClassNotFoundException.
                tryLoadingFromJavaseLoader = true;
            }

            if (tryLoadingFromJavaseLoader) {
                try {
                    clazz = javaseLoader.loadClass(name);
                    if (clazz != null) {
                        if (resolve)
                            resolveClass(clazz);
                        return (clazz);
                    }
                } catch (ClassNotFoundException e) {
                    // Ignore
                }
            }
```
这里有一个小的优化点，不是直接调用loadClass去加载类，而是先尝试使用getResource获取资源，如果资源存在再进行类加载，这样防止触发ClassNotFoundException。

# 如果有需要则代理给父类加载器

这一步首先会判断tomcat启动的时候是否设置了delgate属性，如果设置了则直接代理给父加载器，这里也就是URLClassLoader。如果没有设置这个属性，则会判断当前需要加载的类是否满足某些特性，如果满足也代理给父类加载器。这里的特性是指加载的类在下面的package中，
```java
javax.el.*
javax.servlet.*
javax.websocket.*
javax.securit.auth.message.*
org.apache.el
org.apache.catalina.*
org.apache.jasper.*
org.apache.juli.*
org.apache.tomcat.*
org.apache.naming.*
org.apache.coyote.*
```
这所以将这些类代理给父类是因为这些类在tomcat启动后应该是唯一的，如果由WebappClassLoader来加载则会出现下面的情况，
> 某个tomcat的包被打包到了lib目录下，但是这个包与当前tomcat版本不兼容，这时候就会出现WebappClassLoader首先加载该类导致系统出现诡异的问题。

# 搜索本地目录加载
这个步骤会扫描当前context下面的lib目录以及classes目录，查找相应的类。这个步骤在大多数情况下加载的都是与业务相关联的类，与tomcat框架关联不是很大。加载完的类会被记录到当前类加载的内部缓存中，也就是之前提到的resourceEntries中。这样提高了类加载的效率。。

# 代理给父类加载器
如果之前的步骤都加载不到该类，则尝试使用父类加载器进行加载。如果还是加载不到，则最后返回ClassNotFoundException。

# 总结

类的加载过程大致如上所示，接下来我们讨论下这么做究竟有什么好处？设想一下下面的场景，
> context A和context B都是用了a.jar，但是两个版本不一致。如果采用传统的类加载方式会出现下面的问题  
> time1: context A加载classC，然后将这个加载操作被代理给了父类加载器，如果父类加载器加载成功返回。  
> time2: context B也加载classC，然后将这个加载操作代理给父类加载器，由于context A和context B使用到的父类加载器是同一个那么返回的是  time1阶段加载的那个类，但是由于业务需要context A和context B使用的classC应该是不同版本才对，返回相同的结果导致了结果的不确定性，甚至  > 系统不可用

通过上面的例子我们了解到对于非jdk和tomcat框架的类，一般都采用子加载器优先而不是父加载器优先的原因。

