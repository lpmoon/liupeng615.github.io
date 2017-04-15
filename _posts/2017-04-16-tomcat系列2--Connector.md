tomcat作为web容器，在处理高并发连接方面有着优异的性能，而这都与其精巧的代码架构及实现有关。tomcat将处理请求的整个组件抽象为一个connector，这个connector在tomcat的整个生命周期中负责接受连接，处理请求，返回结果等。比如下面的代码定义了一个处理http请求的connector，

```

    <Connector port="8080" protocol="HTTP/1.1"

               connectionTimeout="20000"

               redirectPort="8443" />

```

通过上面的代码，我们队connector有了一些初步的印象，

1. connector需要定义一个端口，这个端口用于监听接受请求

2. connector需要定义一个协议，这个协议用于解析请求处理请求

3. connector还需要定义一个连接超时时间



除了这些connector还需要什么，它具体是怎么工作的呢？为了弄明白这些，需要深入到connector的具体实现中，



```

public class Connector extends LifecycleMBeanBase  {

    public Connector(String protocol) {

        setProtocol(protocol);

        // Instantiate protocol handler

        ProtocolHandler p = null;

        try {

            Class<?> clazz = Class.forName(protocolHandlerClassName);

            p = (ProtocolHandler) clazz.newInstance();

        } catch (Exception e) {

            log.error(sm.getString(

                    "coyoteConnector.protocolHandlerInstantiationFailed"), e);

        } finally {

            this.protocolHandler = p;

        }



        if (!Globals.STRICT_SERVLET_COMPLIANCE) {

            URIEncoding = "UTF-8";

            URIEncodingLower = URIEncoding.toLowerCase(Locale.ENGLISH);

        }

    }

}

```

connector实例化的时候会根据配置的protocolHandlerClassName初始化protocolHandler，这个handler用于后面的协议处理。connector同其所在的容器catalina一样都继承了LifecycleBase，因此它的整个生命周期分为了以下的几步，

1. init

2. start

3. stop

4. destroy



connector的init过程通过initInternal实现，initInternal最主要的工作是调用protocolHandler的init方法，protocolHandler在connector实例化的时候被初始化。protocolHandler的初始化主要分为两步，

1. 注册mbean

2. 调用endpoint的初始化



由于tomcat支持多协议，并且同一协议也有多种实现，所以protocolHandler在tomcat也有多种实现，为了更好的理解代码，我们选取Http11Nio2Protocol作为我们跟踪代码的主线。Http11Nio2Protocol选取Nio2Endpoint作为其endpoint，Nio2Endpoint的init操作在其抽象的父类中实现，主要调用bind操作完成端口的绑定，



```

    public void init() throws Exception {

        if (bindOnInit) {

            bind();

            bindState = BindState.BOUND_ON_INIT;

        }

    }

```

具体的绑定操作由具体的实现类来定义，以下是Nio2Endpoint的实现，

```

   public void bind() throws Exception {



        // Create worker collection

        if ( getExecutor() == null ) {

            createExecutor();

        }

        if (getExecutor() instanceof ExecutorService) {

            threadGroup = AsynchronousChannelGroup.withThreadPool((ExecutorService) getExecutor());

        }

        // AsynchronousChannelGroup currently needs exclusive access to its executor service

        if (!internalExecutor) {

            log.warn(sm.getString("endpoint.nio2.exclusiveExecutor"));

        }



        serverSock = AsynchronousServerSocketChannel.open(threadGroup);

        socketProperties.setProperties(serverSock);

        InetSocketAddress addr = (getAddress()!=null?new InetSocketAddress(getAddress(),getPort()):new InetSocketAddress(getPort()));

        serverSock.bind(addr,getBacklog());



        // Initialize thread count defaults for acceptor, poller

        if (acceptorThreadCount != 1) {

            // NIO2 does not allow any form of IO concurrency

            acceptorThreadCount = 1;

        }



        // Initialize SSL if needed

        initialiseSsl();

    }

```

1. 创建线程池

2. 根据设置的监听地址以及端口和backlog数量初始化监听端口，这里用到了nio.2的一些东西

3. 重置acceptor线程数

4. 初始化ssl



到这里connector的初始化完成，下面就会进入到启动阶段。connector的启动阶段依然延续了init的老套路，connector->protocolHanlder->endpoint，最终还是会进入到Nio2Endpoint的startInternal, 

```

    /**

     * Start the NIO2 endpoint, creating acceptor.

     */

    @Override

    public void startInternal() throws Exception {



        if (!running) {

            allClosed = false;

            running = true;

            paused = false;



            processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,

                    socketProperties.getProcessorCache());

            nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,

                    socketProperties.getBufferPool());



            // Create worker collection

            if ( getExecutor() == null ) {

                createExecutor();

            }



            initializeConnectionLatch();

            startAcceptorThreads();

        }

    }

```

1. 创建存放SocketProcessorBase和Nio2Channel的stack。前者用于处理消息。

2. 再次确认线程池是否初始化，如果没有则重新初始化

3. 初始化最大连接限制，其内部实现采用了tomcat自己实现的LimitLatch。

4. 启动acceptor线程，这里只会启动一个线程，这是因为在init阶段对acceptor线程数做了处理，如果用户配置的线程数大于1，则重置为1。这是因为NIO.2不允许并行操作。



启动acceptor的线程的操作如下所示，

```

    protected final void startAcceptorThreads() {

        int count = getAcceptorThreadCount();

        acceptors = new Acceptor[count];



        for (int i = 0; i < count; i++) {

            acceptors[i] = createAcceptor();

            String threadName = getName() + "-Acceptor-" + i;

            acceptors[i].setThreadName(threadName);

            Thread t = new Thread(acceptors[i], threadName);

            t.setPriority(getAcceptorThreadPriority());

            t.setDaemon(getDaemon());

            t.start();

        }

    }

```

acceptor线程的具体定义在Nio2Endpoint中，其具体实现比较长，我们需要做一下分解

* 如果线程被暂停，则需要进入内层的while循环，为了防止线程空转耗费cpu，线程每50ms被唤醒一次重新检查线程是否被启动

```

                // Loop if endpoint is paused

                while (paused && running) {

                    state = AcceptorState.PAUSED;

                    try {

                        Thread.sleep(50);

                    } catch (InterruptedException e) {

                        // Ignore

                    }

                }

```

* 如果线程被终止，则跳出线程的外层循环

```

 if (!running) {

                    break;

                }

```

* 对计数器进行加一操作，接受新的连接

```

                    //if we have reached max connections, wait

                    countUpOrAwaitConnection();



                    AsynchronousSocketChannel socket = null;

                    try {

                        // Accept the next incoming connection from the server

                        // socket

                        socket = serverSock.accept().get();

                    } catch (Exception e) {

                        // We didn't get a socket

                        countDownConnection();

                        if (running) {

                            // Introduce delay if necessary

                            errorDelay = handleExceptionWithDelay(errorDelay);

                            // re-throw

                            throw e;

                        } else {

                            break;

                        }

                    }

                    // Successful accept, reset the error delay

                    errorDelay = 0;

```

这里需要注意的是计数器操作在接受连接之前，这样才能保证连接数的精准，设想一次如果先调用accept，然后才对计数器操作加一，在高并发的情况下很可能导致服务器大量连接。同时如果接收连接异常，需要恢复计数器，也就是减一操作。

* 处理新的连接

```

                    // Configure the socket

                    if (running && !paused) {

                        // setSocketOptions() will hand the socket off to

                        // an appropriate processor if successful

                        if (!setSocketOptions(socket)) {

                            closeSocket(socket);

                       }

                    } else {

                        closeSocket(socket);

                    }

```

具体的操作在setSocketOptions中，

```

    protected boolean setSocketOptions(AsynchronousSocketChannel socket) {

        try {

            // 步骤1

            socketProperties.setProperties(socket);

            // 步骤2

            Nio2Channel channel = nioChannels.pop();

            if (channel == null) {

                SocketBufferHandler bufhandler = new SocketBufferHandler(

                        socketProperties.getAppReadBufSize(),

                        socketProperties.getAppWriteBufSize(),

                        socketProperties.getDirectBuffer());

                if (isSSLEnabled()) {

                    channel = new SecureNio2Channel(bufhandler, this);

                } else {

                    channel = new Nio2Channel(bufhandler);

                }

            }

            Nio2SocketWrapper socketWrapper = new Nio2SocketWrapper(channel, this);

            channel.reset(socket, socketWrapper);

            socketWrapper.setReadTimeout(getSocketProperties().getSoTimeout());

            socketWrapper.setWriteTimeout(getSocketProperties().getSoTimeout());

            socketWrapper.setKeepAliveLeft(Nio2Endpoint.this.getMaxKeepAliveRequests());

            socketWrapper.setSecure(isSSLEnabled());

            socketWrapper.setReadTimeout(getSoTimeout());

            socketWrapper.setWriteTimeout(getSoTimeout());

            // Continue processing on another thread

            // 步骤3

            return processSocket(socketWrapper, SocketEvent.OPEN_READ, true);

        } catch (Throwable t) {

            ExceptionUtils.handleThrowable(t);

            log.error("",t);

        }

        // Tell to close the socket

        return false;

    }

```

首先会设置socket的一些属性，然后封装成为socketWrapper，socketWrapper包含了多个handler，分别用于读取数据完毕，写入数据完毕后的回调操作。最后调用processSocket开始处理。processSocket会将socketWrapper和event(此时为SocketEvent.OPEN_READ)封装成SocketProcessorBase，放入到线程池中执行。

```

public boolean processSocket(SocketWrapperBase<S> socketWrapper,

            SocketEvent event, boolean dispatch) {

        try {

            if (socketWrapper == null) {

                return false;

            }

            SocketProcessorBase<S> sc = processorCache.pop();

            if (sc == null) {

                sc = createSocketProcessor(socketWrapper, event);

            } else {

                sc.reset(socketWrapper, event);

            }

            Executor executor = getExecutor();

            if (dispatch && executor != null) {

                executor.execute(sc);

            } else {

                sc.run();

            }

        } catch (RejectedExecutionException ree) {

            getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);

            return false;

        } catch (Throwable t) {

            ExceptionUtils.handleThrowable(t);

            // This means we got an OOM or similar creating a thread, or that

            // the pool and its queue are full

            getLog().error(sm.getString("endpoint.process.fail"), t);

            return false;

        }

        return true;

    }

```

SocketProcessorBase的初始化由createSocketProcessor完成，而createSocketProcessor的具体定义在Nio2Endpoint中，

```

    @Override

    protected SocketProcessorBase<Nio2Channel> createSocketProcessor(

            SocketWrapperBase<Nio2Channel> socketWrapper, SocketEvent event) {

        return new SocketProcessor(socketWrapper, event);

    }

```

SocketProcessor中最主要的步骤是获取handler，然后调用handler的process方法。

```

                if (handshake == 0) {

                    SocketState state = SocketState.OPEN;

                    // Process the request from this socket

                    if (event == null) {

                        state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);

                    } else {

                        state = getHandler().process(socketWrapper, event);

                    }

                    if (state == SocketState.CLOSED) {

                        // Close socket and pool

                        closeSocket(socketWrapper);

                        if (running && !paused) {

                            if (!nioChannels.push(socketWrapper.getSocket())) {

                                socketWrapper.getSocket().free();

                            }

                        }

                    } else if (state == SocketState.UPGRADING) {

                        launch = true;

                    }

                } else if (handshake == -1 ) {

                    closeSocket(socketWrapper);

                    if (running && !paused) {

                        if (!nioChannels.push(socketWrapper.getSocket())) {

                            socketWrapper.getSocket().free();

                        }

                    }

                }

```

这个handler是ConnectionHandler的实例，handler的process代码很长，这里删除一些对主流程影响不是很大的代码，

```

      @Override

        public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {

            if (getLog().isDebugEnabled()) {

                getLog().debug(sm.getString("abstractConnectionHandler.process",

                        wrapper.getSocket(), status));

            }

            if (wrapper == null) {

                // Nothing to do. Socket has been closed.

                return SocketState.CLOSED;

            }



            S socket = wrapper.getSocket();



            Processor processor = connections.get(socket);

            if (getLog().isDebugEnabled()) {

                getLog().debug(sm.getString("abstractConnectionHandler.connectionsGet",

                        processor, socket));

            }



            if (processor != null) {

                // Make sure an async timeout doesn't fire

                getProtocol().removeWaitingProcessor(processor);

            } else if (status == SocketEvent.DISCONNECT || status == SocketEvent.ERROR) {

                // Nothing to do. Endpoint requested a close and there is no

                // longer a processor associated with this socket.

                return SocketState.CLOSED;

            }



            ContainerThreadMarker.set();



            try {

				// 省略代码。。。。。。

                if (processor == null) {

                    processor = recycledProcessors.pop();

                    if (getLog().isDebugEnabled()) {

                        getLog().debug(sm.getString("abstractConnectionHandler.processorPop",

                                processor));

                    }

                }

                if (processor == null) {

                    processor = getProtocol().createProcessor();

                    register(processor);

                }



                processor.setSslSupport(

                        wrapper.getSslSupport(getProtocol().getClientCertProvider()));



                // Associate the processor with the connection

                connections.put(socket, processor);



                SocketState state = SocketState.CLOSED;

                do {

                    state = processor.process(wrapper, status);



                    if (state == SocketState.UPGRADING) {

						// 省略代码。。。。。。

                    }

                } while ( state == SocketState.UPGRADING);



                if (state == SocketState.LONG) {

                    // In the middle of processing a request/response. Keep the

                    // socket associated with the processor. Exact requirements

                    // depend on type of long poll

                    longPoll(wrapper, processor);

                    if (processor.isAsync()) {

                        getProtocol().addWaitingProcessor(processor);

                    }

                } else if (state == SocketState.OPEN) {

                    // In keep-alive but between requests. OK to recycle

                    // processor. Continue to poll for the next request.

                    connections.remove(socket);

                    release(processor);

                    wrapper.registerReadInterest();

                } else if (state == SocketState.SENDFILE) {

                    // Sendfile in progress. If it fails, the socket will be

                    // closed. If it works, the socket either be added to the

                    // poller (or equivalent) to await more data or processed

                    // if there are any pipe-lined requests remaining.

                } else if (state == SocketState.UPGRADED) {

                    // Don't add sockets back to the poller if this was a

                    // non-blocking write otherwise the poller may trigger

                    // multiple read events which may lead to thread starvation

                    // in the connector. The write() method will add this socket

                    // to the poller if necessary.

                    if (status != SocketEvent.OPEN_WRITE) {

                        longPoll(wrapper, processor);

                    }

                } else {

                    // Connection closed. OK to recycle the processor. Upgrade

                    // processors are not recycled.

                    connections.remove(socket);

                    if (processor.isUpgrade()) {

                    	// 省略代码。。。。。。

                    } else {

                        release(processor);

                    }

                }

                return state;

            } catch(java.net.SocketException e) {

                // SocketExceptions are normal

                getLog().debug(sm.getString(

                        "abstractConnectionHandler.socketexception.debug"), e);

            } catch (java.io.IOException e) {

                // IOExceptions are normal

                getLog().debug(sm.getString(

                        "abstractConnectionHandler.ioexception.debug"), e);

            } catch (ProtocolException e) {

                // Protocol exceptions normally mean the client sent invalid or

                // incomplete data.

                getLog().debug(sm.getString(

                        "abstractConnectionHandler.protocolexception.debug"), e);

            }

            // Future developers: if you discover any other

            // rare-but-nonfatal exceptions, catch them here, and log as

            // above.

            catch (Throwable e) {

                ExceptionUtils.handleThrowable(e);

                // any other exception or error is odd. Here we log it

                // with "ERROR" level, so it will show up even on

                // less-than-verbose logs.

                getLog().error(sm.getString("abstractConnectionHandler.error"), e);

            } finally {

                ContainerThreadMarker.clear();

            }



            // Make sure socket/processor is removed from the list of current

            // connections

            connections.remove(socket);

            release(processor);

            return SocketState.CLOSED;

        }

```

1. 根据socket获取Processor，这里的Processors涉及到具体的协议。

2. 如果获取成功，则将Processor从waitingProcessors中移除，否则进入3

3. 如果当前status为DISCONNECT或者ERROR，则返回SocketState.CLOSED，process结束；否则计入到4

4. 如果Processor为空，则创建新的Processor，并且注册Processor。

5. 调用Processor的process方法进行连接处理

6. 根据返回的state进行处理



其中主流程在Processor的process方法中，主要涉及到了读取数据，进行协议解析，然后根据之前在初始化阶段解析的分发规则将消息分配给对应的应用进行处理，然后写入返回值。这其中还包含了读取数据完毕后以及写入数据完毕后的handler的回调（在socketWrapper初始化的时候注册的）。



再次回到SocketProcessorBase中，获取到ConnectionHandler的返回值后，如果返回值是SocketState.CLOSED，则调用closeSocket(socketWrapper)关闭socket，在closeSocket中会将当前的连接数减一。



至此一个完整的连接完成了。

