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



# Spring Cloud学习





# # 问题

## 为什么gRPC选择HTTP2？