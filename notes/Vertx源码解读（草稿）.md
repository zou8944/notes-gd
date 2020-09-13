## Vertx源码探究

这里我们希望按照顺序，解读如下几个部分源码

- Vertx-core
  - Vertx
  - EventBus
  - Context
  - EventLoop
- HTTP和TCP部分（也属于Vertx-Core）
- HA（也属于Vertx-core）
- Web部分
  - Web Server
  - Web Client
- 两个第三方库
  - PostgreSQL Client
  - OAuth 2

## 概述

通过本次研究，我们希望能够明确回答出如下几个问题

1. 一个vertx应用中，vertx实例是否一定只有一个
2. 是如何实现一个handler仅对应一个event-loop的，即只对应一个线程的
3. 网络库、DNS库等，使用vertx实例做了什么，为什么创建它们时总是需要一个vertx实例
4. vertx为我们带来了什么？市面上还有哪些其它的库有同等能力？为什么我们选择了vertx呢？
5. vertx集群的实现原理？

## 核心类

VertxFactory - VertxFactoryImpl

Vertx - VertxImpl

Context - ContextImpl

EventBus - EventBusImpl

VertxThread - 



#### Vertx.vertx()干了什么？

最终执行了VertxImpl.vertx()方法，创建了VertxImpl对象。将EventBus的start标志置位，然后返回创建的VertxImpl对象。

创建VertxImpl时干了什么？

```java
// 创建closeHooks，CloseHooks维护了一个Closeable的Set，可向其中添加、移除任务，还有执行所有钩子的run方法啦。
closeHooks = new CloseHooks(log);
// 创建线程阻塞检查器，它启动一个名为vertx-blocked-thread-checker的定时器，
checker = new BlockedThreadChecker(options.getBlockedThreadCheckInterval(), options.getBlockedThreadCheckIntervalUnit(), options.getWarningExceptionTime(), options.getWarningExceptionTimeUnit());
// 指定一个EventLoop最长可以连续执行多久
maxEventLoopExTime = options.getMaxEventLoopExecuteTime();
maxEventLoopExecTimeUnit = options.getMaxEventLoopExecuteTimeUnit();
// 创建EventLoop线程工厂，主要用于指定线程名称和线程阻塞检测器
eventLoopThreadFactory = new VertxThreadFactory("vert.x-eventloop-thread-", checker, false, maxEventLoopExTime, maxEventLoopExecTimeUnit);
// 创建EventLoopGroup，它又实际创建了NioEventLoopGroup，它是Netty的组件。一个EventLoopGroup，就是一个EventLoop组。在Netty中，一个EventLoop是线程和IO的结合，一个EventLoop始终绑定在同一个线程上。
eventLoopGroup = transport.eventLoopGroup(Transport.IO_EVENT_LOOP_GROUP, options.getEventLoopPoolSize(), eventLoopThreadFactory, NETTY_IO_RATIO);
// 创建一个acceptor EventLoopGroup，创建方式和上面类似。
ThreadFactory acceptorEventLoopThreadFactory = new VertxThreadFactory("vert.x-acceptor-thread-", checker, false, options.getMaxEventLoopExecuteTime(), options.getMaxEventLoopExecuteTimeUnit());
acceptorEventLoopGroup = transport.eventLoopGroup(Transport.ACCEPTOR_EVENT_LOOP_GROUP, 1, acceptorEventLoopThreadFactory, 100);
// 创建worker线程池
ExecutorService workerExec = new ThreadPoolExecutor(workerPoolSize, workerPoolSize,
      0L, TimeUnit.MILLISECONDS, new LinkedTransferQueue<>(),
      new VertxThreadFactory("vert.x-worker-thread-", checker, true, options.getMaxWorkerExecuteTime(), options.getMaxWorkerExecuteTimeUnit()));
PoolMetrics workerPoolMetrics = metrics != null ? metrics.createPoolMetrics("worker", "vert.x-worker-thread", 	options.getWorkerPoolSize()) : null;
workerPool = new WorkerPool(workerExec, workerPoolMetrics);
// 创建inertnal阻塞线程池
ExecutorService internalBlockingExec = Executors.newFixedThreadPool(options.getInternalBlockingPoolSize(),
        new VertxThreadFactory("vert.x-internal-blocking-", checker, true, options.getMaxWorkerExecuteTime(), options.getMaxWorkerExecuteTimeUnit()));
internalBlockingPool = new WorkerPool(internalBlockingExec, internalBlockingPoolMetrics);
// 创建文件解析器，在FileSystem中有使用，进行文件操作时使用的是java nio
this.fileResolver = new FileResolver(options.getFileSystemOptions());
// 创建地址解析器，推测在DNS解析时会用到
this.addressResolver = new AddressResolver(this, options.getAddressResolverOptions());
// 创建发布管理器，用于发布Verticle
this.deploymentManager = new DeploymentManager(this);
if (options.getEventBusOptions().isClustered()) {
    // 创建集群管理器和集群的EventBus
    this.clusterManager = getClusterManager(options);
    this.eventBus = new ClusteredEventBus(this, options, clusterManager);
} else {
    // 创建本地EventBus
    this.clusterManager = null;
    this.eventBus = new EventBusImpl(this);
}
// 创建sharedData，允许你在整个应用中共享你的数据，包括集群范围内
this.sharedData = new SharedDataImpl(this, clusterManager);
```



#### 线程阻塞超时2秒如何侦测？

```
Vertx运行的线程为自定义类VertxThread，继承BlockedThreadChecker.Task。其中的maxExecTime()表示该线程允许最大执行时间。在BlockedThreadChecker启动的定时器中，每隔一段时间检测一次线程启动时间(VertxThread.startTime)到now()的间隔，超过2秒则报警。
public interface Task {
    long startTime();
    long maxExecTime();
    TimeUnit maxExecTimeUnit();
}
这也好理解，每当VertxThread被调度执行时，更新一次startTime，如果执行太久，startTime来不及更新，自然会报警。
```

疑问：如果是因为长时间没有调度到一个VertxThread线程导致startTime未得到及时更新，应该怎么办。



#### BlockedThreadChecker和VertxThread如何绑定？

VertxThreadFactory最能说明问题，一个VertxThreadFactory对应一个BlockedThreadChecker，每当启动一个线程，就将该线程加入checker，checker存了一个线程Map，这样checker就能定期扫描该Map了。

```java
public Thread newThread(Runnable runnable) {
    VertxThread t = new VertxThread(runnable, prefix + threadCount.getAndIncrement(), worker, maxExecTime, maxExecTimeUnit);
    // Vert.x threads are NOT daemons - we want them to prevent JVM exit so embededd user doesn't
    // have to explicitly prevent JVM from exiting.
    if (checker != null) {
    	checker.registerThread(t, t);
    }
    addToMap(t);
    // I know the default is false anyway, but just to be explicit-  Vert.x threads are NOT daemons
    // we want to prevent the JVM from exiting until Vert.x instances are closed
    t.setDaemon(false);
    return t;
}
```



acceptorEventLoopGroup的作用是什么？



internalBlockingPool的作用是什么？



DeploymentManager是如何工作的？



ClusterManager是如何工作的？ClusteredEventBus是如何工作的？



EventBus本地版和集群版是如何工作的？



SharedData的工作原理如何？



#### vertx.eventBus().send("", "")干了什么？

```java
//最终会来到sendOrPubInternal，首先创建一个用于回复的HandlerRegistration，然后创建OutboundDeliveryContext，调用其next方法
public <T> void sendOrPubInternal(MessageImpl message, DeliveryOptions options,
                                  Handler<AsyncResult<Message<T>>> replyHandler) {
    checkStarted();
    HandlerRegistration<T> replyHandlerRegistration = createReplyHandlerRegistration(message, options, replyHandler);
    OutboundDeliveryContext<T> sendContext = new OutboundDeliveryContext<>(message, options, replyHandlerRegistration);
    sendContext.next();
}
// createReplyHandlerRegistration方法创建了__vertx.reply.xxx地址的响应HandlerRegistration
private <T> HandlerRegistration<T> createReplyHandlerRegistration(MessageImpl message,
                                                                  DeliveryOptions options,
                                                                  Handler<AsyncResult<Message<T>>> replyHandler) {
    if (replyHandler != null) {
        long timeout = options.getSendTimeout();
        String replyAddress = generateReplyAddress();
        message.setReplyAddress(replyAddress);
        Handler<Message<T>> simpleReplyHandler = convertHandler(replyHandler);
        HandlerRegistration<T> registration =
            new HandlerRegistration<>(vertx, metrics, this, replyAddress, message.address, true, replyHandler, timeout);
        registration.handler(simpleReplyHandler);
        return registration;
    } else {
        return null;
    }
}
protected String generateReplyAddress() {
    return "__vertx.reply." + Long.toString(replySequence.incrementAndGet());
}
// OutboundDeliveryContext类接收了消息和响应HandlerRegistration，调用next，如下。其中的iter多半是拦截器，暂时不用管。核心在sendOrPub(this)和sendReply(this, replierMessage)
@Override
public void next() {
    if (iter.hasNext()) {
        Handler<DeliveryContext> handler = iter.next();
        try {
            if (handler != null) {
                handler.handle(this);
            } else {
                next();
            }
        } catch (Throwable t) {
            log.error("Failure in interceptor", t);
        }
    } else {
        if (replierMessage == null) {
            sendOrPub(this);
        } else {
            sendReply(this, replierMessage);
        }
    }
}
// 定义io.vertx.core.eventbus.impl.EventBusImpl#sendOrPub，再定位到io.vertx.core.eventbus.impl.EventBusImpl#deliverMessageLocally,最终来到io.vertx.core.eventbus.impl.EventBusImpl#deliverMessageLocally
// 这里的关键由两个地方：一是点对点的实现——再handlerMap中找到指定地址的handlers，只取第一个进行处理；还有发布订阅的实现——对在一个地址注册的handlers全部处理；第二个关键点是消息发送的方法deliverToHandler(msg, holder)
protected ReplyException deliverMessageLocally(MessageImpl msg) {
    msg.setBus(this);
    ConcurrentCyclicSequence<HandlerHolder> handlers = handlerMap.get(msg.address());
    if (handlers != null) {
        if (msg.isSend()) {
            //Choose one
            HandlerHolder holder = handlers.next();
            if (metrics != null) {
                metrics.messageReceived(msg.address(), !msg.isSend(), isMessageLocal(msg), holder != null ? 1 : 0);
            }
            if (holder != null) {
                deliverToHandler(msg, holder);
                Handler<AsyncResult<Void>> handler = msg.writeHandler;
                if (handler != null) {
                    handler.handle(Future.succeededFuture());
                }
            }
        } else {
            // Publish
            if (metrics != null) {
                metrics.messageReceived(msg.address(), !msg.isSend(), isMessageLocal(msg), handlers.size());
            }
            for (HandlerHolder holder: handlers) {
                deliverToHandler(msg, holder);
            }
            Handler<AsyncResult<Void>> handler = msg.writeHandler;
            if (handler != null) {
                handler.handle(Future.succeededFuture());
            }
        }
        return null;
    } else {
        ... ...
    }
}
// 最终的处理函数如下：创建InboundDeliveryContext，在HandlerHolder的context环境下运行其next方法：
private <T> void deliverToHandler(MessageImpl msg, HandlerHolder<T> holder) {
    // Each handler gets a fresh copy
    MessageImpl copied = msg.copyBeforeReceive();
    DeliveryContext<T> receiveContext = new InboundDeliveryContext<>(copied, holder);

    if (metrics != null) {
        metrics.scheduleMessage(holder.getHandler().getMetric(), msg.isLocal());
    }

    holder.getContext().runOnContext((v) -> {
        try {
            receiveContext.next();
        } finally {
            if (holder.isReplyHandler()) {
                holder.getHandler().unregister();
            }
        }
    });
}
// next方法啥也没干，直接将message传入目标handler了
@Override
public void next() {
    if (iter.hasNext()) {
        // ... 拦截器迭代，忽略
    } else {
        holder.getHandler().handle(message);
    }
}
```

上面还解释了为什么在一个context下注册的handler最终都还是在该context线程下执行，因为这个context在注册时被保存了下来，执行时直接在其下执行。

#### vertx.eventBus().consumer("", handler)干了什么？

```java
@Override
public <T> MessageConsumer<T> consumer(String address, Handler<Message<T>> handler) {
    Objects.requireNonNull(handler, "handler");
    MessageConsumer<T> consumer = consumer(address);
    consumer.handler(handler);
    return consumer;
}
// 往里进一步
@Override
public <T> MessageConsumer<T> consumer(String address) {
    checkStarted();
    Objects.requireNonNull(address, "address");
    return new HandlerRegistration<>(vertx, metrics, this, address, null, false, null, -1);
}
// 重点在HandlerRegistration，收集地址后，开启超时回复定时器。
public HandlerRegistration(Vertx vertx, EventBusMetrics metrics, EventBusImpl eventBus, String address,
                               String repliedAddress, boolean localOnly,
                               Handler<AsyncResult<Message<T>>> asyncResultHandler, long timeout) {
    this.vertx = vertx;
    this.metrics = metrics;
    this.eventBus = eventBus;
    this.address = address;
    this.repliedAddress = repliedAddress;
    this.localOnly = localOnly;
    this.asyncResultHandler = asyncResultHandler;
    if (timeout != -1) {
        timeoutID = vertx.setTimer(timeout, tid -> {
            if (metrics != null) {
                metrics.replyFailure(address, ReplyFailure.TIMEOUT);
            }
            sendAsyncResultFailure(new ReplyException(ReplyFailure.TIMEOUT, "Timed out after waiting " + timeout + "(ms) for a reply. address: " + address + ", repliedAddress: " + repliedAddress));
        });
    }
}
// 最上面的consumer.handler(handler);调用了HandlerRegistration的handler方法，如下。可以看到最终是在eventBus上调用了注册方法。
@Override
public synchronized MessageConsumer<T> handler(Handler<Message<T>> h) {
    if (h != null) {
        synchronized (this) {
            handler = h;
            if (registered == null) {
                registered = eventBus.addRegistration(address, this, repliedAddress != null, localOnly);
            }
        }
        return this;
    }
    this.unregister();
    return this;
}
// 最终来到了EventBus的addRegistration方法。在addLocalRegistration中，创建了HandlerHolder，并将其加入EventBus的成员变量handlerMap，然后返回创建的HandlerHolder
protected <T> HandlerHolder<T> addRegistration(String address, HandlerRegistration<T> registration,
                                               boolean replyHandler, boolean localOnly) {
    Objects.requireNonNull(registration.getHandler(), "handler");
    LocalRegistrationResult<T> result = addLocalRegistration(address, registration, replyHandler, localOnly);
    addRegistration(result.newAddress, address, replyHandler, localOnly, registration::setResult);
    return result.holder;
}

总之，调用vertx.eventBus().consumer("", handler)，仅仅是向EventBusImpl的handlerMap变量中加入了一个HandlerHolder。而该holder持有新创建的HandlerRegistration
```

至此，我们还是没有看到注册的内容如何和线程绑定在一起的。甚至没有看到vertx是什么时候启动的。

那换个思路，看看vertx.deployVerticle()做了什么？



#### vertx.deployVerticle()做了什么？

```java
// 经过跟踪，我们到达的第一个有意义的方法是io.vertx.core.impl.DeploymentManager#doDeployVerticle
// 可以看到如下关键代码：verticle是在executeBlocking中创建的。通过verticleFactory创建后，调用doDeploy进行发布。
vertx.<Verticle[]>executeBlocking(createFut -> {
    try {
        Verticle[] verticles = createVerticles(verticleFactory, identifier, options.getInstances(), cl);
        createFut.complete(verticles);
    } catch (Exception e) {
        createFut.fail(e);
    }
}, res -> {
    if (res.succeeded()) {
        doDeploy(identifier, options, parentContext, callingContext, completionHandler, cl, res.result());
    } else {
        // Try the next one
        doDeployVerticle(iter, res.cause(), identifier, options, parentContext, callingContext, cl, completionHandler);
    }
});

// 我们来到方法io.vertx.core.impl.DeploymentManager#doDeploy，里面能够清楚地看到verticle发布的代码。代码太长就不贴了。
// 能够收到如下几个方面的信息
// 1. 发布是由父子层级关系的，在一个Verticle中发布另一个Verticle，则该Verticle就是子Verticle。发布层级关系由Deployment类控制
// 2. 整个发布流程为
//	- 调用vertx.createEventLoopContext(deploymentID, pool, conf, tccl)方法创建一个Context
//  - 向该Context中添加Deployment对象，将verticle对象塞入Deploydment对象。这样方便在Context关闭时调用verticle的stop方法
//  - 调用context.runOnContext()方法执行verticle的方法：init、start
//  - 发布完成
```

至此，我们看到了verticle的创建是通过executeBlocking创建的，真实的发布是通过Context.runOnContext执行的，此时start中的语句可执行。

**还有一点很重要，正是因为针对每个verticle，vertx都会创建一个新的Context，因此每个verticle之间是相互独立的(一个Context代表了一个EventLoop线程。)**

**而正是因为一整个verticle的内容都通过Context.runOnContext注册运行，所以它们才会始终都在一个线程上执行，并且执行顺序从上到下，不存在多线程竞争问题。**可以证明

```kotlin
fun main() {
  val vertx = Vertx.vertx()
  vertx.runOnContext {
    vertx.eventBus().consumer<String>("hello") {
      println(Thread.currentThread().name)
      println(it.body())
      it.reply("world")
    }
    vertx.eventBus().consumer<String>("hello") {
      println(Thread.currentThread().name)
      println(it.body())
      it.reply("world")
    }
    vertx.eventBus().consumer<String>("hello") {
      println(Thread.currentThread().name)
      println(it.body())
      it.reply("world")
    }
    vertx.eventBus().publish("hello", "hello")
  }
}
```

输出内容是

```bash
vert.x-eventloop-thread-0
hello
vert.x-eventloop-thread-0
hello
vert.x-eventloop-thread-0
hello
```

而如果没有runOnContext，则采用round-robin的方式分配线程，并且彼此之间并发执行。

```kotlin
fun main() {
  val vertx = Vertx.vertx()
  vertx.eventBus().consumer<String>("hello") {
    println(Thread.currentThread().name)
    println(it.body())
    it.reply("world")
  }
  vertx.eventBus().consumer<String>("hello") {
    println(Thread.currentThread().name)
    println(it.body())
    it.reply("world")
  }
  vertx.eventBus().consumer<String>("hello") {
    println(Thread.currentThread().name)
    println(it.body())
    it.reply("world")
    vertx.eventBus().publish("hello", "hello")
  }
}
```

```bash
vert.x-eventloop-thread-2
vert.x-eventloop-thread-0
vert.x-eventloop-thread-1
hello
hello
hello
```

现在的问题是：在start中注册的consumer是什么时候执行呢？



#### vertx.eventBus().consumer()注册的内容是如何加入到EventLoop中的？EventLoop是什么时候启动的？也就是说，EventBus是如何与EventLoop结合的？

```
vertx启动时，创建了eventLoopGroup和EventBusImpl，其中，EventLoopGroup是Netty的组件、EventBusImpl接收一个vertx实例作为成员变量。
因此，初始化时看，应该是EventBus包含了vertx实例，vertx实例包含了EventLoop

操作EventBus().consumer()，只是向其handlerMap添加了一个HandlerHolder，等待被执行。

操作EventBus().send()，会将对应的HandlerHolder从handlerMap中拿出来(依据地址取出)，然后再holder预存的context上将消息传递给目标handler。调用了context.runOnContext()方法。

那context.runOnContext()方法又是如何与EventLoop绑定的呢？
```



### Context

context代表的是一个handler的执行上下文。

一个context就是一个event-loop context，并被绑定在特定的event loop线程，因此在一个context的代码执行永远发生在同一个event-loop线程。

worker verticle略有不同，它的context运行的线程是从worker pool中取的一个，是不一定的。

#### vertx.runOnContext()

```java
// 这里很明显了，就是getOrCreateContext()和context.runOnContext()的结合。
public void runOnContext(Handler<Void> task) {
    ContextImpl context = getOrCreateContext();
    context.runOnContext(task);
}
```

#### context.runOnContext()

```java
// 看到只调用了一个executeAsync()
@Override
public void runOnContext(Handler<Void> task) {
    try {
        executeAsync(task);
    } catch (RejectedExecutionException ignore) {
        // Pool is already shut down
    }
}
// 这里就能看到Vertx的底了，它直接将任务提交给了netty的eventLoop
void executeAsync(Handler<Void> task) {
    nettyEventLoop().execute(() -> executeTask(null, task));
}
```

#### vertx.getOrCreateContext()

```java
// 该方法获取当前线程绑定的Context对象，如果当前线程没有绑定，则创建一个新的EventLoopContext，创建过程和下面说的一样。
public ContextImpl getOrCreateContext() {
    ContextImpl ctx = getContext();
    if (ctx == null) {
        // We are running embedded - Create a context
        ctx = createEventLoopContext(null, null, new JsonObject(), Thread.currentThread().getContextClassLoader());
    }
    return ctx;
}
public ContextImpl getContext() {
    ContextImpl context = (ContextImpl) ContextImpl.context();
	......
}
// 定位到这里：我们这里暂时不管FastThreadLocalThread，看到是VertxThread时就返回；不是时就返回null。
public static Context context() {
    Thread current = Thread.currentThread();
    if (current instanceof VertxThread) {
        return ((VertxThread) current).getContext();
    } else if (current instanceof FastThreadLocalThread) {
        return holderLocal.get().ctx;
    }
    return null;
}
```

从上面可以看出一个点，如果在普通线程如主线程中调用vertx.getOrCreateContext()，将会创建一个Context，在该Context下运行的代码将在新的EventLoop线程，与主线程就没有关系了。

#### vertx.createEventLoopContext

该方法来自verticle的发布方法，对每个verticle创建了一个Event Loop上下文

```java
// 它就创建了一个EventLoopContext类
@Override
public EventLoopContext createEventLoopContext(String deploymentID, WorkerPool workerPool, JsonObject config, ClassLoader tccl) {
    return new EventLoopContext(this, internalBlockingPool, workerPool != null ? workerPool : this.workerPool, deploymentID, config, tccl);
}
// EventLoopContext类在实例化时有一个关键的地方：getEventLoop的调用
protected ContextImpl(VertxInternal vertx, WorkerPool internalBlockingPool, WorkerPool workerPool, String deploymentID, JsonObject config, ClassLoader tccl) {
    this(vertx, getEventLoop(vertx), internalBlockingPool, workerPool, deploymentID, config, tccl);
}
// 它从vertx的EventLoopGroup获取一个EventLoop
private static EventLoop getEventLoop(VertxInternal vertx) {
    EventLoopGroup group = vertx.getEventLoopGroup();
    if (group != null) {
        return group.next();
    } else {
        return null;
    }
}
```

可以很明显地看到，Context和EventLoop进行了绑定，一个Context持有一个EventLoop对象。

#### Vertx.currentContext()

```java
static @Nullable Context currentContext() {
    return factory.context();
}
// 最终还是调用了ContextImpl.context()，即获取当前线程的Context，如果当前线程不是EventLoop线程，将获取到空。
@Override
public Context context() {
    return ContextImpl.context();
}
```

