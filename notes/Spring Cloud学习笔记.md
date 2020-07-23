# Spring Cloud学习笔记

## 微服务为什么一定要用Spring Cloud

简单来说，微服务化的核心就是将传统的一站式应用根据业务拆分成一个一个的服务，而微服务在这个基础上要更彻底地去耦合（不再共享DB、KV，去掉重量级ESB），并且强调DevOps和快速演化。

将业务拆分成多个单独的服务，那么他们之间的管理就需要一个中心组件来操作。负责服务注册、服务发现、管理服务的可用性等。

首先映入眼帘的是Dubbo，它能够实现微服务的基本功能：注册服务、远程调用服务、服务调用统计等；但是他的问题在于

- 其中的服务注册依赖于Zookeeper之类的第三方组件；
- DUBBO采用RPC调用，使得服务提供方和调用方代码定义上高度依赖，不方便。

从[官方例子](https://github.com/apache/dubbo-samples/blob/master/java/dubbo-samples-api/src/main/java/org/apache/dubbo/samples/api/GreetingsService.java)可以看到Dubbo的基本使用方法:

定义

## 什么是RPC的调用方式

RPC，即远程过程调用。它对应的是本地过程调用：本地过程调用，即在当前环境本地调用方法。就是我们常写的方法调用。

而远程过程调用，在调用形式上，还是方法调用，只不过调用的方法并不在本地，而是位于远端的RPC服务器。

不同于本地调用，远程调用的过程参数是需要先序列化再反序列化的，因此性能上会比本地差。但因为序列化后走的TCP协议，因此会比走Http协议的Restful API性能更好。

不过RPC的定义和概念也就仅限于此了。其实说来，RPC可以建立再TCP上，也可以建立再TCP上。实现时如何传入，则完全看具体实现

- Dubbo自定义报文，直接走TCP协议
- gRPC则直接基于HTTP2协议

## RPC和gRPC区别

上面说了，RPC并没有规定底层实现走TCP还是HTTP，而gRPC就是RPC的一种具体实现，规定走HTTP2协议。

## 微服务之间调用走RPC还是Restful API有关系吗？

这个问题是看[这篇文章](https://zhuanlan.zhihu.com/p/94755320)时提出的，他的语境是Dubbo，Dubbo的RPC是自定义的报文，走TCP协议。而Restful API走HTTP1.1协议，而HTTP1.1协议有太多无用的信息，性能上肯定比Dubbo自定义的报文差一大截。

## Dubbo和SpringCloud的对比

![preview](https://pic1.zhimg.com/v2-ec2653e66a9dc0bcc2b475694ec0c6e8_r.jpg)



## 为什么需要微服务

概念上说，微服务就是将原本一个独立的系统拆分成多个小型服务，这些服务在各自的进程中运行，他们之间通过HTTP进行通信写作。

使用微服务的原因，是在在业务发展初期，单体应用的开发、测试、部署会比较方便，但是随着业务的扩展，单体应用内各业务高度耦合等弊端就显现出来了，维护成本变得越来越高，到最终变得难以控制。而通过服务拆分，在修改一个服务时不会影响其他服务的运行，且可以依据需要对部分服务扩充或缩减。

微服务带来的好处是服务间解耦、伸缩灵活、复杂度可控；坏处是分布式系统的复杂性较高，运维较为困难。

## 为什么使用Spring Cloud

在微服务的构建中，需要解决各种问题，如服务治理、分布式配置管理、批量任务、服务跟踪等，每个技术问题都有对应的解决方案和开源框架，自己组合这些开源框架时就像自己组装一台电脑，有可能因为自己不了解某个框架而导致整个系统运行不畅。而Spring Cloud则相当于买品牌机，提供了微服务架构所需的方方面面的技术，并且测试通过，我们只管使用。

前面说到的Dubbo，就只是一个服务治理框架而已。

如果架构师够厉害，能够自己选择一套技术，那Spring Cloud就不适用。

# Spring Cloud学习

## Spring Boot的配置真的简单吗？

默认情况下自动配置，如果不想使用默认配置，则要修改，那么又回到配置上来了。有很多潜规则配置。

- SpEL表达式的学习负担
- 多环境配置（看起来是灵活的一点）
- 参数端口
- 其他
- **配置可以分布在各处，有一个固定的加载顺序，容易踩坑。**

这些配置随着项目的发展会将这些配置外部化，使用到的工具有Spring Cloud Config

## Spring将配置从xml转移到各个配置类上真的好吗？

Spring Boot为了改善繁杂的配置内容，采用包扫描和自动化配置的机制来加载原本集中在XML文件中的内容。这样代码变得简洁了，但是分析整个系统的变得非常困难。

这一点通过actuator可以一定程度予以解决。

## SpringBoot和Spring actuator

Spring actuator能够采集系统的统计信息，通过接口的形式暴露出来。甚至包括metrics信息。

## 服务治理 - Spring Cloud Eureka

服务治理是微服务最为核心和基础的模块。主要是用来实现各个微服务实例的自动化注册与发现。至于为什么需要服务注册与发现，可以想象没有他我们要做什么，我们要为每个节点配置与他相互通信的节点的各项信息，在这些节点变化时还要手动维护他们。如果节点非常多，人是不可能做的好的。

有了服务中心后，每次新节点都向他注册自己，服务中心维护节点数据；当其他节点需要时，向服务中心查询，就能得到想要的结果。

Eureka高可用原理：将自己作为服务向其他服务中心注册自己（需要手动配置），形成一组互相注册的服务注册中心，实现服务清单的相互同步，这样服务注册中心就有多个实例了，且每个实例的服务清单是同步的。

服务端配置高可用原理后，服务提供方需要将注册中心指向所有注册中心。

```properties
eureka.client.serviceUrl.defaultZone=http://peer1:1111/eureka/,http://peer2:1111/eureka/
```

代码参考：https://github.com/zou8944/spring-cloud-demo/commit/6fa3979c7ce284b65a6191b60197bce44fb41de6

## 客户端负载均衡 - Spring Cloud Ribbon

基于Netflix Ribbon实现。它只是一个工具类框架，不需要单独部署。

在使用中，其核心在于Spring自己提供的RestTemplate类与@LoadBalanced注解的结合使用。

RestTemplate类与@LoadBalanced注解的工作原理如下：

LoadBalancerAutoConfiguration自动配置：将RestTemplate注入到当前类，方便操作，创建LoadBalancerInterceptor，再将RestTemplate和LoadBalancerInterceptor进行绑定，使得RestTemplate每次发送请求时都必须经过LoadBalancerInterceptor，实现负载均衡。则关键逻辑都在LoadBalancerInterceptor.intercept()中了。在拦截器的方法中，又调用了LoadBalancerClient的实现类RibbonLoadBalancerClient进行负载均衡。

通过分析RibbonLoadBalancerClient，可以知负载均衡操作是在ILoadBalancer实现类实现的。具体内容有兴趣再下去看。

**重试机制**

Eureka是一个强调CAP中的AP特性的框架，即可用性与可靠性；Zookeeper是强调CP特性的框架，即一致性与可靠性。因此再Eureka中，如果注册中心网络发生故障时，它不会提出所有服务实例；而是会因为丢失心跳而触发保护机制，注册中心还是会保留所有服务实例，即使其中又腹胀节点，这样可以保证大多数的服务仍然可以正常工作。

由于上述原因，或者服务提出的延迟，都有可能引起服务调用端调用目标服务的超时，为了增强对这类问题的容错，我们在服务调用时候需要加入一些重试机制。此时需要增加如下配置即可。

```properties
# 开启重试
spring.cloud.loadbalancer.retry.enabled=true
# 断路器超时时间要大于Ribbon的超时时间，不然不会触发重试
hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=10000
hello-service.ribbon.ConnectTimeout=250
hello-service.ribbon.ReadTimeout=1000
hello-service.ribbon.OkToRetryOnAllOperations=true
hello-service.ribbon.MaxAutoRetriesNextServer=2
hello-service.ribbon.MaxAutoRetries=1
```

## 服务容错保护 - Spring Cloud Hystrix

为什么需要断路器？

如果没有断路器，针对一个服务的请求频繁出现超时，在微服务系统中会导致连锁超时，当请求并发量过大时会存在大量等待中的任务，最终将整个系统严重拖慢，甚至崩溃。有了断路器，如果一个服务不可用，则直接报错或进行服务降级，不会影响整体服务的性能。

关于Hytrix的工作原理，请有需要时再去研究，这里我们只是体验一下。

Hytrix除了实现断路器，还有许多额外实用的功能

- 缓存：在断路器层缓存目标服务的返回结果，省去了专门弄一个缓存层。
- 请求合并：HytrixCollapser可以将请求合并，减少通信消耗。合并的是在一个默认10ms的时间窗对同意以来服务的多个请求进行整合，并批量发起请求的功能。有点像Redis的批量执行命令。

**断路器监控**

Hystrix-dashboard提供了对断路器的监控，可以对集群监控，也可以对单个实例监控，还可以将多个实例通过turbine聚合进行监控。

甚至可以在聚合之后将监控信息存入消息队列再由turbine拉取进行显示，完成监控信息的异步收集。

代码参考：https://github.com/zou8944/spring-cloud-demo/commit/2e3a06fba984da08a87ae2ab1bcf44b31db16357

## 声明式服务调用 - Spring Cloud Feign

Feign是对Ribbon和Hystrix的进一步封装，使得只需要通过声明的方式即可达到调用的目的。替代原本每次都要直接操作RestTemplate的方式。

代码参考：https://github.com/zou8944/spring-cloud-demo/commit/c61cfa2efca8471b6b12fee0e2a3027110a6c5bd

利用Feign的继承特性重构后的结果

代码参考：https://github.com/zou8944/spring-cloud-demo/commit/8b511a033a94626da9a16527b3adb171463c15fc

当然，Feign是对Ribbon和Hystrix的进一步封装，当然会提供他们的配置，具体请参考书籍，这里不提供。

## API网关服务 - Spring Cloud Zuul

将服务集群对外暴露，对外服务这块内容，也称为**边缘服务**

API网关解决的问题

- 自动维护对外路由和集群内的服务实例列表
- 解决多个微服务都需要的冗余前置操作，如校验等

Netflix Zuul如何解决这个问题的

- 针对自动维护路由和实例列表的需求；Zuul将自己注册为eureka的一个服务，从而获取集群中的所有实例，然后将服务名自动映射成ContextPath，使得其几乎不需要额外配置就能满足基本需求

  使用上

  - 手动配置：只需要配置route的前缀和serviceId的对应关系即可

  - 自动配置：在网关项目中引入Spring-Eureka后，他会为Eureka中的每个服务都自动创建一个默认规则，规则的path都会使用serviceId作为前缀。

    此时通过zuul.ignored-service配置需要忽略的服务

  具体路由的转发、匹配规则、前缀等有更加自由的定制方式，这里不深究。

- 针对冗余前置操作，我们将这些前置操作独立成单独的服务，然后通过Zuul进行统一调配。Zuul提供一套过滤器机制，开发者创建过滤器，指定哪些规则的请求需要执行校验逻辑，只有校验通过的才会被真正转发到具体的微服务接口，不然就返回错误提示。

  使用上是通过继承ZuulFilter来实现的。

  过滤器在Zuul中非常重要，几乎所有功能都是通过针对不同生命周期阶段的过滤器来实现的，包括最为核心的路由功能。

Zuul的额外功能

- 本身包含了Hystrix和Ribbon，因此能够熔断和负载均衡。
- 动态加载配置：Zuul作为全天候服务，需要不掉线，配置需要热更新。这主要通过Spring Config实现的

## 分布式配置中心 - Spring Cloud Config

Spring配置中心主要包含以下几个元素

- 远程Git仓库
- 本地Git仓库
- Config Server
- 具体微服务

工作原理：在Config Server指定远程仓库抵制，配置中心将其拉取到本地（通过git clone命令），每次客户端向配置中心请求时，先尝试获取远程Git仓库的内容；如果远程仓库无法获取，则返回本地仓库的内容

Spring Config的实际配置非常灵活，比如可配置多个仓库，配置项获取方式多样，支持Git、SVN、本地仓库、本地文件系统等。此外还有属性覆盖、安全保护等功能。还提供对单独的属性进行加解密的功能（适用于属性本身是密码的情况）

服务中心的高可用：比较好的方式是将配置中心当成众多微服务中的一个，方便维护和管理。

## 消息总线 - Spring Cloud Bus

消息总线的意思：构建一个消息主题，让系统中所有微服务实例都连接上来，由于主题中的消息会被所有服务实例监听和消费，所以称为消息总线。

消息代理中间件：消息代理是一种消息验证、传输、路由的架构模式。在应用程序之间起到通信调度并最小化应用之间的依赖，使得应用程序可以高校解耦通信过程。消息代理的核心是消息路由，即接收和发送程序。目前主流的开源产品供大家使用

- ActiveMQ
- Kafka
- RabbitMQ
- RocketMQ

目前Spring仅支持AMQP和Kafka

**AMQP**



代码参考：https://github.com/zou8944/spring-cloud-demo/commit/81eb816fe5a8fc0bd0221f9e72da07877619a03b

**结合Spring Cloud Bus和Spring Config进行配置自动更新**

上一节介绍了如何进行Config的热更新，但每次都要触发所有客户端去重新拉取新配置，比较麻烦。可以配置Git的webhook，在每次修改后，向消息总线发送拉取消息，再由每个客户端订阅该总线，每次收到消息时就自动拉取配置。完成真正的自动化更新。

## 消息驱动的微服务 - Spring Cloud Stream

Spring Cloud Stream为消息中间件提供了个性化的自动化配置实现，可以减少开发人员对消息中间件的使用复杂度。

这是一个高度抽象的东西，我连最基本的消息队列都不怎么会使用，这玩意儿可以暂时忽略。

代码参考：https://github.com/zou8944/spring-cloud-demo/commit/497722522d119948469965ac56571cd2bfbf5f35

## 分布式服务跟踪 - Spring Cloud Sleuth

链路跟踪的原理：

- 标记一条链路：再请求发送到分布式系统入口处时，为其创建一个唯一的跟踪标识，再分布式系统内部流转时，始终保持该标识的流转。通过该标识可以将所有相关的日志联系起来，叫做Trace ID
- 统计各处理单元延迟：当请求到达各服务组件时，再分配一个唯一标识给当前组件，叫做Span ID，卡Span ID出现的头尾时间，即为当前组件耗时。

Spring Cloud Sleuth加入依赖后，会自动为当前应用构建跟踪机制。例如通过RestTemplate发起的请求。它是把生成的Trace ID等放在请求头的。

还需要考虑的问题是日志采集率。详细记录的日志由海量，不可能全采，因此需要配一个采样率。Sleuth支持。

此外采集到的大量日志还需要日志分析工具，Sleuth支持接入Logstach和Zipkin

# Spring Boot优缺点总结

## 优点

- 继承做得非常好，启动一个功能往往只需要引入依赖，增加注解，修改配置即可使用。

## 缺点

- starter组织了一大堆依赖，在初次引入依赖时要下载很久（在我本地半个小时）;光引入spring-boot-starter这一项都用了
- 一切按照注解来，使得代码逻辑非常破碎，学习成本较高。

# Spring的特点

- SpEL表达式，如果使用Spring的话很有用。但我觉得这又是一种学习负担。

- 配置太多，即使spring boot也会有很多潜规则配置，一不小心很容易掉进坑里。

- 配置太多，想要使用自己不熟悉的配置時要嘗試半天。比如要在测试中注入配置文件，有如下几种方式

  - SpringBootTest
  - ActiveProfiles
  - TestPropertySource
  - 。。。

  为了解决测试目标需要环境变量而找不到的问题，找了半个小时都没找到解决方法。想着都来气。


# # 问题

## 为什么gRPC选择HTTP2？



## Spring Boot中的嵌入式Tomcat，在性能上跟得上吗？



## 服务治理、负载均衡、断路器、网关等组件都有哪些替代品？

