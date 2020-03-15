# Netty In Action学习笔记

## 第一部分 Netty基础

- 3.1.2中说到，每个channel会被分配到一个EventLoop上，一个EventLoop始终是在一个线程上执行，也就是说同一个channel的IO操作一直会在一个线程上执行，那么这个线程不就阻塞了吗？这样的话，为什么说EventLoop是异步的呢？

- Netty从很早时就有了，但是很早的时候Java只有IO包，那时候Netty是如何做到异步的呢？

- 反复观看第三章，搞懂Channel、EventLoop、ChannelFutur、ChannelHandler、ChannelPipline等类之间的关系。

- 从3.2.5节中了解Bootstrap的几种类别以及用法

- 看到截止4.3为止，给人的感觉就是Netty仅仅是对基础库的一种包装，其底层并没有使用什么高级的技术或架构。

- Java的NIO从语言包的角度实现了selector方式，使得不会阻塞，那么JVM是什么时候去操作连接，使得他们状态变化的呢？总不可能神奇地自动变化吧。

- Netty的ByteBuf是对Java的ByteBuffer的简化，ByteBuffer的使用实在是太麻烦了。

- ByteBuf这一章可以不细看。

- 从ByteBuf这一章可以看到，ByteBuf是对Java自带Buf的扩展和加强，带有所有ByteBuffer的功能，比如在堆中分配的ByteBuf，在direct memory中分配的ByteBuf，以及ByteBuffer没有的功能，CompositeByteBuf（它可以将多个ByteBuf放在一起）。此外，还有PooledBuf和UnPooledBuf之分，前者会共享Buf对象，后者每次都新创建一个。另外还介绍了几个有用的类，他们提供有用的静态方法，比如：UnPooled、ByteBufUtil类。还有ByteBufAllocator等用于手动控制分配的类。

- @Sharable的意思：代表该ChannelHandler可以被共享，即能够被添加到多个ChannelPipline中

- 

- Netty使用引用计数来判断一个ByteBuf是否可用，因此使用常规的ChannelHandler时需要我们自己保证ByteBuf是否还需要使用，否则很可能发生内存溢出。

  - 类SimpleChannelInboundHandler会在消息被消费（即channelRead0()被调用时）后自动进行释放

  - 手动释放一个消息的方式

    ```java
    ReferenceCountUtil.release(msg);
    ```

  - 疑问：引用计数不是有缺陷吗？为什么Netty还会采用它呢？

  - 总之，如果用户在一个ChannelHandler中处理了消息，并且没有将其传给下一个ChannelHandler，那该消息的释放责任就落在了用户头上，用户必须像上面那样手动释放。

- 关于ChannelPipline

  - 每一个Channel被创建时都会被分配一个新的ChannelPipline。Channel不能新增也不能取消当前的ChannelPipline，这个规则是固定的，开发者不用也不能对此进行干预。
  - 同一个ChannelPipline中既可以添加InboundHandler也可以添加OutboundHandler，但是无论二者添加顺序如何，Netty在请求到来时只会执行InboundHandler，请求出去时也只会执行OutboundHandler
  - ChannelPipline还有一系列主动触发事件的方法。

- 关于ChannelHandlerContext

  - ChannelHandlerContext是在Handler被添加到Pipline时创建的。他们之间的关系可以如下表示

    ![1570156481728](/home/floyd/.config/Typora/typora-user-images/1570156481728.png)

  - 在一个Event发生后，会在一个Pipline中的所有Handler传递下去，下图可以较好地展示这个过程。（<span style="color:red;">这一点认为需要验证一下，不然使用时会造成一定的麻烦</span>）

    ![1570157850559](/home/floyd/.config/Typora/typora-user-images/1570157850559.png)

- 关于ChannelHandler可以同时被多个ChannelPipline同时拥有，但此时该Handler需要使用@Sharable进行注释，表示它能够被重复利用。否则如果被添加给多个Pipline时会触发一个异常。

  - 此外在多个Pipline中使用同一个Handler时还必须注意线程安全性问题，这意味着它最好不要拥有过多的状态。
  - 什么时候需要共享ChannelHandler？一般不需要，而且是不建议使用的。至于使用到的场景，一般都是用户跨Pipline进行数据统计时需要。

- 关于Netty中处理异常的方式

  - 当一个异常从某个ChannelHandler中被抛出时，它会在Pipline中同方向剩下的所有Handler中流转，除非我们进行控制。
  - 如果抛出异常没有被处理，该异常会流转到Pipline的最末尾，并被日志打印出来。
  - 对于ChannelInboundHandler和ChannelOutboundHandler，处理异常的方式略有不同：对于前者，我们只需要重写ExceptionCaught()方法即可；对于后者，我们则需要对返回的ChannelFuture或重写实现的参数的ChannelPromise添加监听方式，添加自己的逻辑。

- 为什么ChannelInboundHandler和ChannelOutboundHandler的异常处理方式不一样呢？

- 根据第七章来看，貌似《Java并发编程实践》还是有必要研究一下的。

- 关于Netty的线程模型

  - Netty 3和Netty 4的线程模型是不一样的

  - EventLoop继承 ScheduledExecutorService。

  - Netty的EventLoop是两个基础API的协同：cnorrency和network。并发上是基于java并发包得来的，使用了线程池技术；网络上以Channel为基础。

    该线程模型上，一个EventLoop时钟绑定在同一个线程上，任务可以马上或定时提交到EventLoop中；根据需要，可以创建多个EventLoop；一个EventLoop也可以服务多个Channel

  - 该模型中，任务和时间执行的是按先进先出的顺序执行的。

- 关于定期执行任务

  - 传统的ScheduledExecutorService 可以做到，但当定期任务过多时会造成较大的性能负担

  - 对此使用EventLoop可以较好地实现。使用

    ```java
    Channel ch = ...;
    ch.eventLoop().schedule(......)
    ```

- 关于Netty的线程管理

  - 如果一个任务在EventLoop中，则排队执行它，如果不在EventLoop中，则将其加入队列。EventLoop是依照顺序一个一个地执行它们的，这也保证了任何线程都能够直接和Channel进行交流，而不管同步措施
  - 每一个EventLoop都单独维护了自己的TaskQueue
  - 和Vertx一样，EventLoop中不能执行长时等待任务，即不能阻塞EventLoop，这会导致队列中其它任务的阻塞。如果实在需要执行阻塞任务，可以使用EventExecutor

- 关于EventLoop的分配

  EventLoop的分配有两种方式，一是传统的阻塞式，另一种是新的异步模式。我们只说新的模式

  - 一个EventLoopGroup中只有几个固定的EventLoop，在group创建时被同时创建。每个EventLoop同时服务多个Channel，一旦一个Channel被分配给一个EventLoop，该Channel的整个生命周期就都在EventLoop中度过，不会改变。

- 关于Bootstrap

  - Bootstrap用于组织和粘合Netty中的各个部分。

  - Channel和EventLoop的类型必须兼容。比如，指定了NioEventLoopGroup，再注册OioSocketChannel就会报IllegalStateException

  - 启动客户端和启动服务端使用的类和方法是不一样的

  - ServerBootstrap启动时会有两个Channel，ServerChannel用于接收来自客户端的请求；将接收到的请求分配给子Channel(该Channel代表绑定了远端的socket)进行处理。

    ![1570181385278](/home/floyd/.config/Typora/typora-user-images/1570181385278.png)

- 关于从一个Channel启动客户端

  - 场景：服务端在某些情况下会作为第三方系统的客户端工作，此时需要从ServerChannel中启动一个客户端
  - 可以使用笨办法，即Bootstrap进行创建，但过于低效
  - 一个较好的方式是将需要创建客户端的Channel的EventLoop直接赋给Bootstrap，可以参考8.4

- 关于ChannelOptions和Attributes的使用

  - 对于ChannelOptions，通过Bootstrap的option方法传入后，会对所有Channel产生效果。可以设置一些低等级的参数，如keep-alive、timeout等
  - AttributeMap是Netty提供的键值对存储工具，可以存储任何数据，并在上下文中共享

- Netty不仅可用于TCP数据传输，也可用于非连接传输，对应的数据类为 xxxDatagramChannel

- Bootstrap能够让我们的类启动，但用完后还是需要关的，尽管可以将一切交给JVM处理，但这不优雅，优雅的方法是

  ```java
  EventLoopGroup.shutdownGracefully();
  ```

- 关于EmbeddedChannel

  - 它有几个特殊的方法，其能够模拟Inbound和Outbound数据，并在Pipline末尾读取数据，查看是否正常，下图形象地描述了这个问题

    ![1570185562194](/home/floyd/.config/Typora/typora-user-images/1570185562194.png)

  - 使用EmbeddedChannel的方法

    ```java
    // 在单元测试中使用，每次能够测试多个ChannelHandler，一次性传递进去就行了
    EmbeddedChannel channel = new EmbeddedChannel(<your handler1>, <your handler2> ... ...);
    channel.witeInbound(...)
    ```

  - 如果想看看实际测试的例子，包括Inbound和Outbound两个方向的，可以参考9.2

## 第二部分 编解码器

- 关于Decoder
  - 当一个Message被encode或decode后，该message将会被自动调用ReferenceUtilCount.release()进行释放。
  - ByteToMessageDecoder是最基本的类，实现decode方法可逐字节读取，缺点是有时接收到的字节数可能没有达到预期
  - ReplayingDecoder是ByteToMessageDecoder的子类，解决了网络传输字节因分包被拆开的问题。但并非所有操作都支持
  - MessageToMessageDecoder，将一种消息转化成另一种消息
- 关于Encoder
  - MessageToByteEncoder，将消息转换成字节流
  - MessageToMessageEncoder，将一种消息转换为另一种消息
- 关于codec
  - 上面将编码器和解码器分开，但有时需要将编解码功能在一个类中一并管理，此时codec就有它的效果了
  - ByteToMessageCodec，字节和消息之间的编解码关系
  - MessageToMessageCodec，消息和消息之间的编解码关系
  - CombinedChannelDuplexHandler，这是一个神奇的类：将编码器和解码器放在一起定义会丧失一定的可重用性，但该类提供了将两个分开的 Encoder和Decoder 组合起来的方法。见10.4.3

> 如下部分讲述Netty提供的一些编解码器

- SSL/TLS
  - JDK提供了支持SSL/TLS的包，相关的类为SSLContext和SSLEngine。响应地，Netty提供了一个SslHandler，用于进行SSL的编解码，其内部实际上是SSLEngine在工作
  - OpenSsl（对应的类OpenSslEngine）能够提供比JDK的SSLEngine更好的性能，Netty能够被配置为默认使用OpenSSL。
- HTTP/HTTPS
  - Netty对此提供了几个编解码器：HttpRequestEncoder/HttpRequestDecoder、HttpResponseEncoder/HttpResponseDecoder
  - 但由于Http协议会分包，因此需要将其几个包聚集成一个Request进行处理，提供了HttpObjectAggregator
  - HttpContentCompressor提供了HTTP内容的压缩
  - 要变成HTTPS，只需要加上SslHandler在Pipline中即可
  - 使用WebSocket，也提供了一系列类，按照11.2.4所说使用即可
- 处理闲置连接和超时
  - Netty提供IdleStateHandler用于处理连接闲置时间过长的情况；提供ReadTimeoutHandler和WriteTimeoutHandler处理读写超时的情况。
- 大文件传输
  - 提供零复制方案：FileRegion，能够直接将文件从磁盘中进行传输，而不用先读取到内存中
  - 提供常规方案：ChunckedHandler+WriteStreamHandler，先读到内存中再传输
- 序列化和反序列化
  - ObjectDecoder/ObjectEncoder，用于JDK提供的对象的序列化和反序列化
  - MarshallingDecoder/MarshallingEncoder，用于JBoss Marshalling，比JDK提供的序列化快三倍
  - ProtobufDecoder/ProtobufEncoder，按照google提供的protocol buffer进行序列化和反序列化

## 第三部分 网络协议

- real-time web ?
  - real-time web，指的是使用一系列技术，使得用户能够在作者发布信息时尽快获取到该信息，而不是传统的定期检查作者的服务以判断是否有新的信息发布。
- WebSocket
  - 这是用于解决客户端与服务器双向传输问题而产生的新的协议，目前大多数浏览器都支持该协议。
  - 该协议基于HTTP，首先使用HTTP进行所谓的升级握手，完成后再使用WebSocket进行传输
  - Netty提供了一系列解决方案用于解决WebSocket服务端的问题
