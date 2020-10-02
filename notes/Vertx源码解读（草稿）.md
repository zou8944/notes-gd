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



#### acceptorEventLoopGroup的作用是什么？

```java
// 通过追踪调用链可以来到这里：
// io.vertx.core.http.impl.HttpServerImpl#listen(...)
ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(vertx.getAcceptorEventLoopGroup(), availableWorkers);

// Bootstrap是Netty的概念。而acceptorEventLoopGroup也只是一个EventLoopGroup，从中可以看出它只是一个用于Http服务的EventLoop组，用于监听请求。
```

#### internalBlockingPool的作用是什么？

```java
// 创建时有如下语句，可以看到它只是创建了一个固定大小的线程池。
ExecutorService internalBlockingExec = Executors.newFixedThreadPool(options.getInternalBlockingPoolSize(),
        new VertxThreadFactory("vert.x-internal-blocking-", checker, true, options.getMaxWorkerExecuteTime(), options.getMaxWorkerExecuteTimeUnit()));
    PoolMetrics internalBlockingPoolMetrics = metrics != null ? metrics.createPoolMetrics("worker", "vert.x-internal-blocking", options.getInternalBlockingPoolSize()) : null;
    internalBlockingPool = new WorkerPool(internalBlockingExec, internalBlockingPoolMetrics);

// 创建后的internalBlockingPool是在创建EventLoopContext时一起传入的，传入后它成了context的成员
@Override
public EventLoopContext createEventLoopContext(String deploymentID, WorkerPool workerPool, JsonObject config, ClassLoader tccl) {
    return new EventLoopContext(this, internalBlockingPool, workerPool != null ? workerPool : this.workerPool, deploymentID, config, tccl);
}
// 跟踪它来到io.vertx.core.impl.ContextImpl#executeBlockingInternal。
@Override
public <T> void executeBlockingInternal(Handler<Promise<T>> action, Handler<AsyncResult<T>> resultHandler) {
    executeBlocking(action, resultHandler, internalBlockingPool.executor(), internalOrderedTasks, internalBlockingPool.metrics());
}
// 来到最终使用它的地方。可以看到它其实就是在internalBlockingPool线程池提交了一个任务，该任务执行了传入的阻塞操作，并返回结果。注意结果是在原本的Context上运行runOnContext执行的。意味着executeBlocking的过程逻辑执行和结果逻辑执行不在一个线程。后者在调用它的EventLoop中执行。
<T> void executeBlocking(Handler<Promise<T>> blockingCodeHandler,
                         Handler<AsyncResult<T>> resultHandler,
                         Executor exec, TaskQueue queue, PoolMetrics metrics) {
    Object queueMetric = metrics != null ? metrics.submitted() : null;
    try {
        Runnable command = () -> {
            VertxThread current = (VertxThread) Thread.currentThread();
            Object execMetric = null;
            if (metrics != null) {
                execMetric = metrics.begin(queueMetric);
            }
            if (!DISABLE_TIMINGS) {
                current.executeStart();
            }
            Promise<T> promise = Promise.promise();
            try {
                ContextImpl.setContext(this);
                blockingCodeHandler.handle(promise);
            } catch (Throwable e) {
                promise.tryFail(e);
            } finally {
                if (!DISABLE_TIMINGS) {
                    current.executeEnd();
                }
            }
            Future<T> res = promise.future();
            if (metrics != null) {
                metrics.end(execMetric, res.succeeded());
            }
            res.onComplete(ar -> {
                if (resultHandler != null) {
                    runOnContext(v -> resultHandler.handle(ar));
                } else if (ar.failed()) {
                    reportException(ar.cause());
                }
            });
        };
        if (queue != null) {
            queue.execute(command, exec);
        } else {
            exec.execute(command);
        }
    } catch (RejectedExecutionException e) {
        // Pool is already shut down
        if (metrics != null) {
            metrics.rejected(queueMetric);
        }
        throw e;
    }
}
```

上面的代码还有一个重点，注意到`current.executeStart()`和`current.executeEnd()`，分别是对VertxThread的execStart计时。

```java
// 注意这里使用的是System.nanoTime()，因为System.currentMillis()可能在某些系统实现下不准。
public final void executeStart() {
    execStart = System.nanoTime();
}
```

这里还有一个需要注意的地方，即区分executeBlocking和executeBlockingInternal两个方法，前者是在WorkerPool中执行逻辑，后者是在internalBlockingPool中执行逻辑

```java
// 执行器传入的是internalBlockingPool.executor()
@Override
public <T> void executeBlockingInternal(Handler<Promise<T>> action, Handler<AsyncResult<T>> resultHandler) {
    executeBlocking(action, resultHandler, internalBlockingPool.executor(), internalOrderedTasks, internalBlockingPool.metrics());
}
// 执行器传入的是workerPool.executor()
@Override
public <T> void executeBlocking(Handler<Promise<T>> blockingCodeHandler, boolean ordered, Handler<AsyncResult<T>> resultHandler) {
    executeBlocking(blockingCodeHandler, resultHandler, workerPool.executor(), ordered ? orderedTasks : null, workerPool.metrics());
}
```

查看vertx.executeBlocking()代码，它其实就是context.executeBlocking()的封装，因此它是运行在workPool中的

#### DeploymentManager是如何工作的？



ClusterManager是如何工作的？ClusteredEventBus是如何工作的？



EventBus本地版和集群版是如何工作的？



#### SharedData的工作原理如何？

所有关于共享数据的内容都在io.vertx.core.shareddata包下，核心类是SharedDataImpl。这一部分可是有些门道的。

提供如下三种数据结构

- io.vertx.core.shareddata.impl.LocalAsyncLocks

  异步排他锁，在集群内部有效的锁。其实现的思路如下

  - 维护一个ConcurrentMap，存储锁名和等待该锁的Handler列表
  - 每次新来一个获取锁的请求，向等待列表中加入。并启动定时器开始计算超时，超时后直接回调锁等待超时。

  至此加入等待列表的逻辑完成。然后是锁流转逻辑。采用被动的逻辑，非常节省复杂度。

  - 当等待列表为空时，来一个请求就将锁给它；列表不为空时，仅加入等待列表，不做尝试获取锁的操作。
  - 当一个锁被释放时，再主动将锁给等待列表的下一个请求。这样几乎从来不会出现竞争的情况。

- io.vertx.core.shareddata.impl.AsynchronousCounter

  计数器，增减都是原子操作

- io.vertx.core.shareddata.impl.LocalMapImpl

  本地Map，用于单个实例中共享数据。仅是对ConcurrentMap的包装，没有其它特别之处。他的所有操作都是同步的。

- io.vertx.core.shareddata.impl.LocalAsyncMapImpl

  异步Map，同样是对ConcurrentMap的包装。不同之处在于其value是Holder类，它封装了TTL，实现原理是调用vertx.setTimer设置一个TTL长度的定时器，过期移除。

  ```java
  @Override
  public void put(K k, V v, long timeout, Handler<AsyncResult<Void>> completionHandler) {
      long timestamp = System.nanoTime();
      long timerId = vertx.setTimer(timeout, l -> removeIfExpired(k));
      Holder<V> previous = map.put(k, new Holder<>(v, timerId, timeout, timestamp));
      if (previous != null && previous.expires()) {
          vertx.cancelTimer(previous.timerId);
      }
      completionHandler.handle(Future.succeededFuture());
  }
  ```

  可能有顾虑设置太多定时器不好，但vertx其实是将定时任务加入eventLoop线程去执行了，因此并不会增加成本

  ```java
  public long setTimer(long delay, Handler<Long> handler) {
      return scheduleTimeout(getOrCreateContext(), handler, delay, false);
  }
  private long scheduleTimeout(ContextImpl context, Handler<Long> handler, long delay, boolean periodic) {
      if (delay < 1) {
          throw new IllegalArgumentException("Cannot schedule a timer with delay < 1 ms");
      }
      long timerId = timeoutCounter.getAndIncrement();
      InternalTimerHandler task = new InternalTimerHandler(timerId, handler, periodic, delay, context);
      timeouts.put(timerId, task);
      context.addCloseHook(task);
      return timerId;
  }
  InternalTimerHandler(long timerID, Handler<Long> runnable, boolean periodic, long delay, ContextImpl context) {
      this.context = context;
      this.timerID = timerID;
      this.handler = runnable;
      this.periodic = periodic;
      EventLoop el = context.nettyEventLoop();
      if (periodic) {
          future = el.scheduleAtFixedRate(this, delay, delay, TimeUnit.MILLISECONDS);
      } else {
          future = el.schedule(this, delay, TimeUnit.MILLISECONDS);
      }
      if (metrics != null) {
          metrics.timerCreated(timerID);
      }
  }
  ```

  

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

## 集群工作原理

集群SPI接口

<img src="D:\WorkSpace\notes-gd\notes\Vertx源码解读（草稿）.assets\image-20200920120549143.png" alt="image-20200920120549143" style="zoom:80%;" />

- AsyncMultiMap： 用于集群间数据共享。共享的保证机制由具体集群实现决定
- ChoosableIterable：能够跨集群的迭代器
- ClusterManager：集群管理器，负责维护节点
- NodeListener：节点监听器，监听节点的增删。

### Vertx.clusteredVertx()

```java
// 与Vertx.vertx()相比，都会先实例化VertxImpl类。不同的是这次集群选项默认选中。且创建后会将vertx加入到集群中。
static void clusteredVertx(VertxOptions options, Transport transport, Handler<AsyncResult<Vertx>> resultHandler) {
    VertxImpl vertx = new VertxImpl(options, transport);
    vertx.joinCluster(options, resultHandler);
}
```

而在VertxImpl的实例化中，最大的区别是clusterManager有被赋值；eventBus被实例化为ClusteredEventBus类

```java
if (options.getEventBusOptions().isClustered()) {
    this.clusterManager = getClusterManager(options);
    this.eventBus = new ClusteredEventBus(this, options, clusterManager);
} else {
    this.clusterManager = null;
    this.eventBus = new EventBusImpl(this);
}
```

**来看getClusterManager方法**：查找用户自定义的集群管理器类；没有则到类路径中寻找ClusterManagerFactory

```java
private ClusterManager getClusterManager(VertxOptions options) {
    ClusterManager mgr = options.getClusterManager();
    if (mgr == null) {
        String clusterManagerClassName = System.getProperty("vertx.cluster.managerClass");
        if (clusterManagerClassName != null) {
            // We allow specify a sys prop for the cluster manager factory which overrides ServiceLoader
            try {
                Class<?> clazz = Class.forName(clusterManagerClassName);
                mgr = (ClusterManager) clazz.newInstance();
            } catch (Exception e) {
                throw new IllegalStateException("Failed to instantiate " + clusterManagerClassName, e);
            }
        } else {
            // 这里是关键，ClusterManager只是一个SPI，具体的集群管理器实现需要如Hystrix之类的框架给与。
            mgr = ServiceHelper.loadFactoryOrNull(ClusterManager.class);
            if (mgr == null) {
                throw new IllegalStateException("No ClusterManagerFactory instances found on classpath");
            }
        }
    }
    return mgr;
}
```

从上面可以看出，我们指定集群管理器的方法有三种

- 自己new好集群管理器类，然后传给VertxOptions配置
- 通过系统属性vertx.cluster.managerClass指定类名
- 通过在类路径中引入ClusterManagerFactory相关的库

**再来看ClusteredEventBus构造方法**

```java
// 首先ClusteredEventBus是EventBusImpl的子类
public class ClusteredEventBus extends EventBusImpl {
    // 总体而言多了下面这么多属性
    // 集群管理器
    private final ClusterManager clusterManager;
    // 维护集群内其它节点ID和连接的Map
    private final ConcurrentMap<ServerID, ConnectionHolder> connections = new ConcurrentHashMap<>();
    // 用于发送消息的上下文
    private final Context sendNoContext;

    // EventBus选项
    private EventBusOptions options;
    // 当前节点的副本节点信息：用于高可用
    private AsyncMultiMap<String, ClusterNodeInfo> subs;
    // 副本节点，通过上面的subs可以查到节点信息
    private Set<String> ownSubs = new ConcurrentHashSet<>();
    // 当前节点的ID
    private ServerID serverID;
    // 当前节点的集群信息
    private ClusterNodeInfo nodeInfo;
    // 维护一个网络服务，用户接收其它节点的请求
    private NetServer server;
    
    public ClusteredEventBus(VertxInternal vertx,
                           VertxOptions options,
                           ClusterManager clusterManager) {
    super(vertx);
    this.options = options.getEventBusOptions();
    this.clusterManager = clusterManager;
    this.sendNoContext = vertx.getOrCreateContext();
  }
}
```

**最后我们康康joinCluster方法**

结合下面对vertx-zookeeper的分析，joinCluster方法就是在zookeeper集群中创建了一个节点。

```java
// 调用的关键，在于clusterManager.join方法。暂时忽略对HA的分析。
private void joinCluster(VertxOptions options, Handler<AsyncResult<Vertx>> resultHandler) {
    clusterManager.setVertx(this);
    clusterManager.join(ar1 -> {
        if (ar1.succeeded()) {
            createHaManager(options, resultHandler);
        } else {
            log.error("Failed to join cluster", ar1.cause());
            close(ar2 -> resultHandler.handle(Future.failedFuture(ar1.cause())));
        }
    });
}
```

### 展示一个ClusterManager的实现（Zookeeper）

引入vertx-zookeeper后，可以发现它的实现其实没啥，核心类就一个ZookeeperClusterManager

<img src="D:\WorkSpace\notes-gd\notes\Vertx源码解读（草稿）.assets\image-20200920112850113.png" alt="image-20200920112850113" style="zoom:50%; align:left;" />

我们主要关注两个方面：

- vertx在vertx-zookeeper中干了啥

  纵观整个ZookeeperClusterManager类，对它的使用仅仅是调用了vertx.executeBlocking()方法去执行zookeeper实际的方法

- join方法干了啥

  如下，该方法大部分都是zookeeper创建curator的逻辑，忽略后仅剩下一个addLocalNodeID()方法的调用。该方法创建了一个Zookeeper节点，不做深究。

  ```java
  @Override
  public synchronized void join(Handler<AsyncResult<Void>> resultHandler) {
      vertx.executeBlocking(future -> {
          if (!active) {
              active = true;
  
              lockReleaseExec = Executors.newCachedThreadPool(r -> new Thread(r, "vertx-zookeeper-service-release-lock-thread"));
  
              //The curator instance has been passed using the constructor.
              if (customCuratorCluster) {
                  try {
                      addLocalNodeID();
                      future.complete();
                  } catch (VertxException e) {
                      future.fail(e);
                  }
                  return;
              }
  
              if (curator == null) {
                  retryPolicy = new ExponentialBackoffRetry(
                      conf.getJsonObject("retry", new JsonObject()).getInteger("initialSleepTime", 1000),
                      conf.getJsonObject("retry", new JsonObject()).getInteger("maxTimes", 5),
                      conf.getJsonObject("retry", new JsonObject()).getInteger("intervalTimes", 10000));
  
                  // Read the zookeeper hosts from a system variable
                  String hosts = System.getProperty("vertx.zookeeper.hosts");
                  if (hosts == null) {
                      hosts = conf.getString("zookeeperHosts", "127.0.0.1");
                  }
                  log.info("Zookeeper hosts set to " + hosts);
  
                  curator = CuratorFrameworkFactory.builder()
                      .connectString(hosts)
                      .namespace(conf.getString("rootPath", "io.vertx"))
                      .sessionTimeoutMs(conf.getInteger("sessionTimeout", 20000))
                      .connectionTimeoutMs(conf.getInteger("connectTimeout", 3000))
                      .retryPolicy(retryPolicy).build();
              }
              curator.start();
              while (curator.getState() != CuratorFrameworkState.STARTED) {
                  try {
                      Thread.sleep(100);
                  } catch (InterruptedException e) {
                      if (curator.getState() != CuratorFrameworkState.STARTED) {
                          future.fail("zookeeper client being interrupted while starting.");
                      }
                  }
              }
              nodeID = UUID.randomUUID().toString();
              try {
                  addLocalNodeID();
                  future.complete();
              } catch (Exception e) {
                  future.fail(e);
              }
          }
      }, resultHandler);
  }
  ```

### ClusteredEventBus的启动

先来看start方法，该方法的核心调用链是getAsyncMultiMap->getServerHandler->deliverMessageLocally

```java
// 暂且忽略HA，首先获取集群中所有节点的map，再创建server进行监听。监听结果由getServerHandler产生的方法进行处理。
@Override
public void start(Handler<AsyncResult<Void>> resultHandler) {
    // Get the HA manager, it has been constructed but it's not yet initialized
    HAManager haManager = vertx.haManager();
    setClusterViewChangedHandler(haManager);
    clusterManager.<String, ClusterNodeInfo>getAsyncMultiMap(SUBS_MAP_NAME, ar1 -> {
        if (ar1.succeeded()) {
            subs = ar1.result();
            server = vertx.createNetServer(getServerOptions());

            server.connectHandler(getServerHandler());
            server.listen(asyncResult -> {
                if (asyncResult.succeeded()) {
                    int serverPort = getClusterPublicPort(options, server.actualPort());
                    String serverHost = getClusterPublicHost(options);
                    serverID = new ServerID(serverPort, serverHost);
                    nodeInfo = new ClusterNodeInfo(clusterManager.getNodeID(), serverID);
                    vertx.executeBlocking(fut -> {
                        haManager.addDataToAHAInfo(SERVER_ID_HA_KEY, new JsonObject().put("host", serverID.host).put("port", serverID.port));
                        fut.complete();
                    }, false, ar2 -> {
                        if (ar2.succeeded()) {
                            started = true;
                            resultHandler.handle(Future.succeededFuture());
                        } else {
                            resultHandler.handle(Future.failedFuture(ar2.cause()));
                        }
                    });
                } else {
                    resultHandler.handle(Future.failedFuture(asyncResult.cause()));
                }
            });
        } else {
            resultHandler.handle(Future.failedFuture(ar1.cause()));
        }
    });
}
```

> 小知识：再集群模式下，每个EventBus上注册的handler都由一个ID，集群中维护它们的方式将handler-ID和server-ID存成一个map，以便在消息发送时根据handler-id找到对应的server，再在server上去执行该handler。
>
> 上面的getAsyncMultiMap(SUBS_MAP_NAME...就是将这个map从集群中拿下来。然后维护在本地。
>
> 问：那如何通知其它节点我刚注册了一个handler呢？
>
> 答：getAsyncMultiMap(SUBS_MAP_NAME...获取到的是AsyncMultiMap的子类实现，本身就是一个集群级别的Map。它的实现由具体集群管理库做，不用我们操心。

来看getServerHandler：它将接收到的信息解析，然后调用deliverMessageLocally进行处理。deliverMessageLocally的逻辑上面已经分析过。这里不再赘述。

```java
private Handler<NetSocket> getServerHandler() {
    return socket -> {
        RecordParser parser = RecordParser.newFixed(4);
        Handler<Buffer> handler = new Handler<Buffer>() {
            int size = -1;

            public void handle(Buffer buff) {
                if (size == -1) {
                    size = buff.getInt(0);
                    parser.fixedSizeMode(size);
                } else {
                    ClusteredMessage received = new ClusteredMessage();
                    received.readFromWire(buff, codecManager);
                    if (metrics != null) {
                        metrics.messageRead(received.address(), buff.length());
                    }
                    parser.fixedSizeMode(4);
                    size = -1;
                    if (received.codec() == CodecManager.PING_MESSAGE_CODEC) {
                        // Just send back pong directly on connection
                        socket.write(PONG);
                    } else {
                        deliverMessageLocally(received);
                    }
                }
            }
        };
        parser.setOutput(handler);
        socket.handler(parser);
    };
}
```

至此，我们看到了集群模式的Vertx是如何启动的。以及是如何接收信息的。

接下来，我们来看handler的注册和消息的发送。

### vertx.eventBus().consumer

最终可以追踪到这个地址：io.vertx.core.eventbus.impl.clustered.ClusteredEventBus#addRegistration。它只是将其添加到了subs中。

注意前面说过，subs仅仅是维护handler地址到服务节点的映射，其实真正的处理逻辑还是在对应节点本地。处理完后又执行了一次集群间发送的逻辑。

```java
@Override
protected <T> void addRegistration(boolean newAddress, String address,
                                   boolean replyHandler, boolean localOnly,
                                   Handler<AsyncResult<Void>> completionHandler) {
    if (newAddress && subs != null && !replyHandler && !localOnly) {
        // Propagate the information
        subs.add(address, nodeInfo, completionHandler);
        ownSubs.add(address);
    } else {
        completionHandler.handle(Future.succeededFuture());
    }
}
```

### vertx.eventBus().send

最终追踪到如下地址：io.vertx.core.eventbus.impl.clustered.ClusteredEventBus#sendOrPub。

```java
// 它从subs这个map中获取对应节点信息。在onSubsReceived()中处理
@Override
protected <T> void sendOrPub(OutboundDeliveryContext<T> sendContext) {
    if (sendContext.options.isLocalOnly()) {
        if (metrics != null) {
            metrics.messageSent(sendContext.message.address(), !sendContext.message.isSend(), true, false);
        }
        deliverMessageLocally(sendContext);
    } else if (Vertx.currentContext() == null) {
        // Guarantees the order when there is no current context
        sendNoContext.runOnContext(v -> {
            // 这是重点
            subs.get(sendContext.message.address(), ar -> onSubsReceived(ar, sendContext));
        });
    } else {
        subs.get(sendContext.message.address(), ar -> onSubsReceived(ar, sendContext));
    }
}
```

看如何处理

```java
// 根据地址获取到一个ChoosableIterable，即一个handler可能对应多个节点(高可用配置)。
private <T> void onSubsReceived(AsyncResult<ChoosableIterable<ClusterNodeInfo>> asyncResult, OutboundDeliveryContext<T> sendContext) {
    if (asyncResult.succeeded()) {
        ChoosableIterable<ClusterNodeInfo> serverIDs = asyncResult.result();
        if (serverIDs != null && !serverIDs.isEmpty()) {
            sendToSubs(serverIDs, sendContext);
        } else {
            if (metrics != null) {
                metrics.messageSent(sendContext.message.address(), !sendContext.message.isSend(), true, false);
            }
            deliverMessageLocally(sendContext);
        }
    } else {
        log.error("Failed to send message", asyncResult.cause());
        Handler<AsyncResult<Void>> handler = sendContext.message.writeHandler();
        if (handler != null) {
            handler.handle(asyncResult.mapEmpty());
        }
    }
}
```

在来到这里

```java
// 从众多节点中选择一个，sendRemote
private <T> void sendToSubs(ChoosableIterable<ClusterNodeInfo> subs, OutboundDeliveryContext<T> sendContext) {
    String address = sendContext.message.address();
    if (sendContext.message.isSend()) {
        // Choose one
        ClusterNodeInfo ci = subs.choose();
        ServerID sid = ci == null ? null : ci.serverID;
        if (sid != null && !sid.equals(serverID)) {  //We don't send to this node
            if (metrics != null) {
                metrics.messageSent(address, false, false, true);
            }
            sendRemote(sid, sendContext.message);
        } else {
            if (metrics != null) {
                metrics.messageSent(address, false, true, false);
            }
            deliverMessageLocally(sendContext);
        }
    } else {
        // Publish
        boolean local = false;
        boolean remote = false;
        for (ClusterNodeInfo ci : subs) {
            if (!ci.serverID.equals(serverID)) {  //We don't send to this node
                remote = true;
                sendRemote(ci.serverID, sendContext.message);
            } else {
                local = true;
            }
        }
        if (metrics != null) {
            metrics.messageSent(address, true, local, remote);
        }
        if (local) {
            deliverMessageLocally(sendContext);
        }
    }
}
```

来到这里：得到对应节点的连接，然后通过TCP发送出去。

```java
private void sendRemote(ServerID theServerID, MessageImpl message) {
    // We need to deal with the fact that connecting can take some time and is async, and we cannot
    // block to wait for it. So we add any sends to a pending list if not connected yet.
    // Once we connect we send them.
    // This can also be invoked concurrently from different threads, so it gets a little
    // tricky
    ConnectionHolder holder = connections.get(theServerID);
    if (holder == null) {
        // When process is creating a lot of connections this can take some time
        // so increase the timeout
        holder = new ConnectionHolder(this, theServerID, options);
        ConnectionHolder prevHolder = connections.putIfAbsent(theServerID, holder);
        if (prevHolder != null) {
            // Another one sneaked in
            holder = prevHolder;
        } else {
            holder.connect();
        }
    }
    holder.writeMessage((ClusteredMessage) message);
}
```

### 康康ClusteredEventBus#sendReply

最终来到这里，可以看到由于在响应Message中直接记录了发送过来的节点ID，因此可以通过该节点ID，可以直接调用sendRemote发送。

```java
@Override
protected <T> void sendReply(OutboundDeliveryContext<T> sendContext, MessageImpl replierMessage) {
    clusteredSendReply(((ClusteredMessage) replierMessage).getSender(), sendContext);
}

private <T> void clusteredSendReply(ServerID replyDest, OutboundDeliveryContext<T> sendContext) {
    MessageImpl message = sendContext.message;
    String address = message.address();
    if (!replyDest.equals(serverID)) {
        if (metrics != null) {
            metrics.messageSent(address, false, false, true);
        }
        sendRemote(replyDest, message);
    } else {
        if (metrics != null) {
            metrics.messageSent(address, false, true, false);
        }
        deliverMessageLocally(sendContext);
    }
}
```

总结：通过上面系列源码分析，能够看到，集群只是起到了节点管理和注册信息同步的作用，真正的数据发送和处理是各个节点自己通过TCP发送的。

## 高可用工作原理