试想一下如果出现下面的情况会怎么样:
> 由于某个交换机或者路由器出现了问题，导致某些post大文件的请求堆积在交换机或者路由器上，tomcat的工作线程一直拿不到完整的文件数据。

在这种情况下处理的工作线程长时间闲置等待请求数据，导致可用工作线程数变少，从而影响了整个服务器的性能。这时候会影响那些正常的请求。

再试想一下下面的情况:
> 有人蓄意攻击，在建立连接后，只发送了请求的method name过来。

和上面的情况一样，工作线程也会卡住，不同的是卡住的阶段不同，第一种情况在读取请求体的时候卡住，第二种情况在读取request line（也就是method name，uri，protocol等）的时候。

针对上面可能出现的异常，tomcat设置了多个timeout参数，用于控制各个阶段等待的最大时长。

# connectionTimeout
官方文档的解释如下
> The number of milliseconds this Connector will wait, after accepting a connection, for the request URI line to be presented. Use a value of -1 to indicate no (i.e. infinite) timeout. The default value is 60000 (i.e. 60 seconds) but note that the standard server.xml that ships with Tomcat sets this to 20000 (i.e. 20 seconds). Unless disableUploadTimeout is set to false, this timeout will also be used when reading the request body (if any).

可以看出这个参数主要用在读取request line阶段，默认情况下为60s，如果设置的为-1则表示没有超时时间。同时这个参数在
```disableUploadTimeout``` 为true的情况下，还作用于读取请求体的阶段。

该参数在tomcat源码中的使用如下所示，
```
wrapper.setReadTimeout(wrapper.getEndpoint().getSoTimeout());
```
这个设置的timeout主要会在两个阶段用到
1. 客户端建立连接后长时间没有数据到达tomcat，tomcat的poller线程每次select数据后会进入到timeout()方法对超过connectionTimeout时间没有收到数据的连接进行超时处理
2. 客户端建立连接后，在connectionTimeout之前发送了一部分数据，poller将该连接的处理交给工作线程池处理。工作线程将读取时间设置为connectionTimeout，读取后续的数据。

这里的soTimeOut就是connectionTimeout。该值的初始化位于Catalina.java的createStartDigester中，
```
digester.addRule("Server/Service/Connector",
                         new SetAllPropertiesRule(new String[]{"executor", "sslImplementationName"}));
```
在SetAllPropertiesRule的begin方法中会通过反射设置connectionTimeOut，
```
    public boolean setProperty(String name, String value) {
        String repl = name;
        if (replacements.get(name) != null) {
            repl = replacements.get(name);
        }
        return IntrospectionUtils.setProperty(protocolHandler, repl, value);
    }
```
通过relacements.get("connectionTimeout")将connectionTimeout替换为了soTimeout，然后调用protocalHandler的setSoTimeout设置超时时间，这个时间会继续传递到endpoint。

通过telnet可以模拟下上面的情况，telnet连接成功后，什么都不做，等着超时。
当connectionTimeout设置为20s的时候，
![http://wx3.sinaimg.cn/large/87f5e2f6gy1fhl0gdsbahj215p03vjrx.jpg](http://wx3.sinaimg.cn/large/87f5e2f6gy1fhl0gdsbahj215p03vjrx.jpg)
当connectionTimeout设置为30s的时候，
![http://wx1.sinaimg.cn/large/87f5e2f6gy1fhl0gdvh08j215s04ijry.jpg](http://wx1.sinaimg.cn/large/87f5e2f6gy1fhl0gdvh08j215s04ijry.jpg)
在经过了connectionTimeout之后tomcat主动关闭连接。

# connectionUploadTimeout
官方文档的解释如下
> Specifies the timeout, in milliseconds, to use while a data upload is in progress. This only takes effect if disableUploadTimeout is set to false.

指定数据上传时的超时时间，只有在disableUploadTimeout设置为false的时候才有效。connectionUploadTimeout的设置过程和connectionTimeout基本一致。处理完request line后，socketWrapper会重新设置读取的超时时间为connectionUploadTimeout，如果disableUploadTimeout为false。
```
                if (!inputBuffer.parseRequestLine(keptAlive)) {
                    if (inputBuffer.getParsingRequestLinePhase() == -1) {
                        return SocketState.UPGRADING;
                    } else if (handleIncompleteRequestLineRead()) {
                        break;
                    }
                }

                if (endpoint.isPaused()) {
                    // 503 - Service unavailable
                    response.setStatus(503);
                    setErrorState(ErrorState.CLOSE_CLEAN, null);
                } else {
					......
                    
                    // 此处重新设置超时时间
                    if (!disableUploadTimeout) {
                        socketWrapper.setReadTimeout(connectionUploadTimeout);
                    }
                }
```

# disableUploadTimeout
这个参数在之前多次提到过，主要用来启用connectionUploadTimeout这个参数。
