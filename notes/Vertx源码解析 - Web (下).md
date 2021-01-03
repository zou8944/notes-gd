# Vertx源码解析 - Web (下)

我们在前面的文章分析了Vertx核心单机部分的源码。今天轮到Vert.x-Web，由于Web的内容比较多，因此分为上下两部分。

- 上：分析Vert.x-Web核心类及其主要能力。
- 下：分析HttpServer的线程调度

上篇已经分析了Web核心类，我们知道了如下几点：

1. Router就是一个Handler，它的调用者传入的是HttpServerRequest类，该类封装了Http请求的所有内容。
2. Router中构建了RoutingContext并开启处理逻辑的流转。
3. RoutingContext启动后的逻辑由用户指定，是用户可操作的部分。RoutingContext启动前的逻辑由Vertx库提供，它负责Http数据的解析并封装成HttpServerRequest，供用户使用。

并留下了如下两个问题，本文给予解答。

1. Vertx库是如何构建HttpServerRequest并调用Router的handle()的
2. 调用Router时线程如何分配？即每次进来的请求线程如何分配

## 前置知识 - Netty网络编程

众所周知，Vertx基于Netty构建，前面我们看到了Vertx是如何将线程调度交给Netty，这次的网络监听也是如此。因此预习一下Netty网络编程是有必要的。

如下是摘自官网的简单例子，予以讲解：

```java
public class DiscardServer {

    public void run() throws Exception {

        // 创建ChannelHandler，用于处理通道数据
        ChannelInboundHandler channelHandler = new ChannelInboundHandlerAdapter() {
            @Override
            public void channelRead(ChannelHandlerContext ctx, Object msg) { 
                // 数据接收处理，这里直接忽略数据
                ((ByteBuf) msg).release(); 
            }

            @Override
            public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) { 
                // 异常处理
                cause.printStackTrace();
                ctx.close();
            }
        };

        // 创建两个EventLoopGroup，前者用于监听连接，后者负责数据处理。
        EventLoopGroup bossGroup = new NioEventLoopGroup(); 
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            // 创建ServerBootstrap，这是Netty的引导类
            ServerBootstrap b = new ServerBootstrap(); // (2)
            // 传入两个EventLoopGroup，前者用于监听连接，后者负责数据处理。
            b.group(bossGroup, workerGroup)
                    // 指定Channel类型
                    .channel(NioServerSocketChannel.class) // (3)
                    // 添加channel处理器，真生的数据处理逻辑在channelHandler中
                    .childHandler(channelHandler);

            // 启动ServerBootstrap
            ChannelFuture f = b.bind(8080).sync(); // (7)

            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        new DiscardServer().run();
    }
}
```

上述代码有两个重点。

- ServerBootstrap需要两个EventLoopGroup，一个用来处理链接，一个用来处理数据。二者关系如下。

![image.png](https://ucc.alicdn.com/pic/developer-ecology/cc27d56addd74e82b6b6b349c7f3769b.png)

- 数据处理逻辑通过childHandler()方法指定。

## 源码分析 - 创建和启动

前文给出的Vertx启动一个服务的例子如下。此次依然以它为例，重点研究createHttpServer()和listen()两个方法

```kotlin
fun main() {
    val vertx = Vertx.vertx()
    val router = Router.router(vertx)
    router.get("/hello")
        .handler { ctx ->
                  println("哈哈哈，我好开心")
                  ctx.next()
                 }
    .failureHandler(ErrorHandler.create())
    val server = vertx.createHttpServer().requestHandler(router)
    server.listen(8080)
}
```

### 创建

关注两个点

- vertx.createHttpServer()做了啥
- requestHandler(router)做了啥

#### vertx.createHttpServer()

跟踪该方法最终可定位到其创建HttpServerImpl对象，调用了其构造方法

```java
public HttpServerImpl(VertxInternal vertx, HttpServerOptions options) {
    // 配置
    this.options = new HttpServerOptions(options);
    // vertx
    this.vertx = vertx;
    // 获取当前上下文并保存，用作创建上下文。核心部分源码分析时说过context一定程度上和线程挂钩
    this.creatingContext = vertx.getContext();
    if (creatingContext != null) {
        // 添加关闭钩子
        creatingContext.addCloseHook(this);
    }
    // ssl工具类
    this.sslHelper = new SSLHelper(options, options.getKeyCertOptions(), options.getTrustOptions());
}
```

关闭钩子将自己添加了进去，HttpServerImpl实现了Closeable，其close方法包含了诸多内容，按照先后顺序，读者可以先跳过它，后面再来看。

```java
// 表面的关闭方法
@Override
public void close(Handler<AsyncResult<Void>> done) {
    ContextInternal context = vertx.getOrCreateContext();
    PromiseInternal<Void> promise = context.promise();
    close(promise);
    if (done != null) {
        promise.future().setHandler(done);
    }
}

// 实际的关闭方法
public synchronized void close(Promise<Void> completion) {
    // 执行请求handler链中的endHandler，即结束handler。
    if (wsStream.endHandler() != null || requestStream.endHandler() != null) {
        Handler<Void> wsEndHandler = wsStream.endHandler();
        wsStream.endHandler(null);
        Handler<Void> requestEndHandler = requestStream.endHandler();
        requestStream.endHandler(null);
        completion.future().setHandler(ar -> {
            if (wsEndHandler != null) {
                wsEndHandler.handle(null);
            }
            if (requestEndHandler != null) {
                requestEndHandler.handle(null);
            }
        });
    }

    ContextInternal context = vertx.getOrCreateContext();
    if (!listening) {
        executeCloseDone(context, completion, null);
        return;
    }
    listening = false;

    // 关闭actualServer
    synchronized (vertx.sharedHttpServers()) {

        if (actualServer != null) {

            actualServer.httpHandlerMgr.removeHandler(
                new HttpHandlers(
                    this,
                    requestStream.handler(),
                    wsStream.handler(),
                    connectionHandler,
                    exceptionHandler == null ? DEFAULT_EXCEPTION_HANDLER : exceptionHandler)
                , listenContext);

            if (actualServer.httpHandlerMgr.hasHandlers()) {
                // The actual server still has handlers so we don't actually close it
                if (completion != null) {
                    executeCloseDone(context, completion, null);
                }
            } else {
                // No Handlers left so close the actual server
                // The done handler needs to be executed on the context that calls close, NOT the context
                // of the actual server
                actualServer.actualClose(context, completion);
            }
        } else {
            executeCloseDone(context, completion, null);
        }
    }
    if (creatingContext != null) {
        creatingContext.removeCloseHook(this);
    }
}
```

####　requestHandler(router)

非常简单，就干了一件事，向requestStream中添加handler

```java
@Override
public synchronized HttpServer requestHandler(Handler<HttpServerRequest> handler) {
    requestStream.handler(handler);
    return this;
}
```

### 启动

关注一个点

- server.listen(8080)做了啥

该方法最终会定位到io.vertx.core.http.impl.HttpServerImpl#listen(io.vertx.core.net.SocketAddress)，它是本次分析的核心

```java
@Override
public Future<HttpServer> listen(SocketAddress address) {
    if (requestStream.handler() == null && wsStream.handler() == null) {
        throw new IllegalStateException("Set request or websocket handler first");
    }
    if (listening) {
        throw new IllegalStateException("Already listening");
    }
    // 获取当前上下文
    listenContext = vertx.getOrCreateContext();
    listening = true;
    String host = address.host() != null ? address.host() : "localhost";
    int port = address.port();
    List<HttpVersion> applicationProtocols = options.getAlpnVersions();
    sslHelper.setApplicationProtocols(applicationProtocols);
    // 获取vertx的sharedHttpServers，它初始为空map，每次调用listen都创建并向其中添加HttpServerImpl，以便多处共享，避免资源浪费
    Map<ServerID, HttpServerImpl> sharedHttpServers = vertx.sharedHttpServers();
    synchronized (sharedHttpServers) {
        this.actualPort = port; // Will be updated on bind for a wildcard port
        id = new ServerID(port, host);
        // 这里调用肯定为空，不为空的情况只有在其它地方对同一个host和port再次创建HttpServer
        HttpServerImpl shared = sharedHttpServers.get(id);
        if (shared == null || port == 0) {
            serverChannelGroup = new DefaultChannelGroup("vertx-acceptor-channels", GlobalEventExecutor.INSTANCE);
            // 创建ServerBootstrap
            ServerBootstrap bootstrap = new ServerBootstrap();
            //  以vertx提前创建好的acceptorEventLoopGroup为bossGroup，以新创建的availableWorkers为workerGroup。
            bootstrap.group(vertx.getAcceptorEventLoopGroup(), availableWorkers);
            applyConnectionOptions(address.path() != null, bootstrap);
            sslHelper.validate(vertx);
            String serverOrigin = (options.isSsl() ? "https" : "http") + "://" + host + ":" + port;
            // 设置channel处理器，不过这里注意，我们的核心业务逻辑并不在childHandler中，而是下面的addHandlers
            bootstrap.childHandler(childHandler(address, serverOrigin));
            addHandlers(this, listenContext);
            try {
                // 其内部调用了bootstrap.bind()启动服务
                bindFuture = AsyncResolveConnectHelper.doBind(vertx, address, bootstrap);
                bindFuture.addListener((GenericFutureListener<io.netty.util.concurrent.Future<Channel>>) res -> {
                    if (!res.isSuccess()) {
                        synchronized (sharedHttpServers) {
                            sharedHttpServers.remove(id);
                        }
                        listening  = false;
                        if (metrics != null) {
                            metrics.close();
                            metrics = null;
                        }
                    } else {
                        Channel serverChannel = res.getNow();
                        if (serverChannel.localAddress() instanceof InetSocketAddress) {
                            HttpServerImpl.this.actualPort = ((InetSocketAddress)serverChannel.localAddress()).getPort();
                        } else {
                            HttpServerImpl.this.actualPort = address.port();
                        }
                        serverChannelGroup.add(serverChannel);
                    }
                });
            } catch (final Throwable t) {
                listening = false;
                return listenContext.failedFuture(t);
            }
            sharedHttpServers.put(id, this);
            actualServer = this;
        } else {
            // 如果HttpServer已经存在，则直接使用即可。
            // Server already exists with that host/port - we will use that
            actualServer = shared;
            this.actualPort = shared.actualPort;
            addHandlers(actualServer, listenContext);
            VertxMetrics metrics = vertx.metricsSPI();
            this.metrics = metrics != null ? metrics.createHttpServerMetrics(options, address) : null;
        }
        Promise<HttpServer> promise = listenContext.promise();
        actualServer.bindFuture.addListener(res -> {
            if (res.isSuccess()) {
                promise.complete(this);
            } else {
                promise.fail(res.cause());
            }
        });
        return promise.future();
    }
}
```

从上面得到，我们的业务核心在addHandlers中

```java
private void addHandlers(HttpServerImpl server, ContextInternal context) {
    server.httpHandlerMgr.addHandler(
        new HttpHandlers(
            this,
            requestStream.handler(),
            wsStream.handler(),
            connectionHandler,
            exceptionHandler == null ? DEFAULT_EXCEPTION_HANDLER : exceptionHandler)
        , context);
}

// server.httpHandlerMgr.addHandler方法
public synchronized void addHandler(T handler, ContextInternal context) {
    // 该context是创建server时netty的上下文
    EventLoop worker = context.nettyEventLoop();
    // 向availableWorkers加入worker，availableWorkers在上面已经给ServerBootStrap做worker了
    availableWorkers.addWorker(worker);
    Handlers<T> handlers = new Handlers<>();
    // 将传入的handler和上下文绑定起来，即该handler将会一直在该上下文的nettyEventLoop执行
    Handlers<T> prev = handlerMap.putIfAbsent(worker, handlers);
    if (prev != null) {
        handlers = prev;
    }
    handlers.addHandler(new HandlerHolder<>(context, handler));
    hasHandlers = true;
}

// 下面是HttpHandlers对象的构造方法
public HttpHandlers(
    HttpServerImpl server,
    Handler<HttpServerRequest> requestHandler,
    Handler<ServerWebSocket> wsHandler,
    Handler<HttpConnection> connectionHandler,
    Handler<Throwable> exceptionHandler) {
    this.server = server;
    this.requestHandler = requestHandler;
    this.wsHandler = wsHandler;
    this.connectionHandler = connectionHandler;
    this.exceptionHandler = exceptionHandler;
}

// 下面是HttpHandlers的handle方法，worker最终会调用他。
@Override
public void handle(HttpServerConnection conn) {
    server.connectionMap.put(conn.channel(), (ConnectionBase) conn);
    conn.channel().closeFuture().addListener(fut -> {
        server.connectionMap.remove(conn.channel());
    });
    // 这是传入的HttpServerRequest的handler
    Handler<HttpServerRequest> requestHandler = this.requestHandler;
    conn.exceptionHandler(exceptionHandler);
    // conn中封装好了HttpServerRequest，处理器传入，准备接收
    conn.handler(requestHandler);
    if (connectionHandler != null) {
        // We hand roll event-loop execution in case of a worker context
        ContextInternal ctx = conn.getContext();
        ContextInternal prev = ctx.beginEmission();
        try {
            connectionHandler.handle(conn);
        } catch (Exception e) {
            ctx.reportException(e);
        } finally {
            ctx.endEmission(prev);
        }
    }
}

// 最后会来到这里：io.vertx.core.http.impl.Http1xServerConnection#handleMessage
public void handleMessage(Object msg) {
    if (msg instanceof HttpRequest) {
        DefaultHttpRequest request = (DefaultHttpRequest) msg;
        if (request.decoderResult() != DecoderResult.SUCCESS) {
            handleError(request);
            return;
        }
        Http1xServerRequest req = new Http1xServerRequest(this, request);
        requestInProgress = req;
        if (responseInProgress != null) {
            // Deferred until the current response completion
            responseInProgress.enqueue(req);
            req.pause();
            return;
        }
        responseInProgress = requestInProgress;
        if (METRICS_ENABLED) {
            reportRequestBegin(req);
        }
        // 这是重点：分配执行
        req.context.dispatch(req, r -> {
            req.handleBegin();
            requestHandler.handle(r);
        });
    } else if (msg == LastHttpContent.EMPTY_LAST_CONTENT) {
        onEnd();
    } else if (msg instanceof HttpContent) {
        onContent(msg);
    } else if (msg instanceof WebSocketFrame) {
        // TODO
        handleWsFrame((WebSocketFrame) msg);
    }
}

// 来到这里：io.vertx.core.impl.EventLoopContext#dispatch(io.vertx.core.impl.AbstractContext, T, io.vertx.core.Handler<T>)
private static <T> void dispatch(AbstractContext ctx, T value, Handler<T> task) {
    EventLoop eventLoop = ctx.nettyEventLoop();
    if (eventLoop.inEventLoop()) {
        if (AbstractContext.context() == ctx) {
            ctx.emit(value, task);
        } else {
            ctx.dispatchFromIO(value, task);
        }
    } else {
        ctx.execute(value, task);
    }
}
```

## 总结

现在来回答上面两个问题

1. Vertx库是如何构建HttpServerRequest并调用Router的handle()的

   调用Netty的网络API，接收数据后封装到Http1xServerConnection，在其内部构建HttpServerRequest，然后分发给Router。

   位置在io.vertx.core.impl.EventLoopContext#dispatch(io.vertx.core.impl.AbstractContext, T, io.vertx.core.Handler\<T>)

2. 调用Router时线程如何分配？即每次进来的请求线程如何分配

   走EventLoop线程，在创建HttpServer那个线程上执行。