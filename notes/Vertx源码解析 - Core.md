# Vert.x源码解析 - Core

> 希望通过本文的解析，让读者了解Vertx的关键部分的实现原理。对诸如如下问题有一个具象的认识。
>
> - Vertx实例的作用？一个应用是否只对应一个Vertx实例？
> - Verticle是一个怎样的存在？
> - 本地模式下消息是如何在EventBus上传输和响应的？
> - EventBus和EventLoop是如何关联起来的？

## 概述

Vert.x是一个事件驱动，基于Netty库构建的高性能应用程序框架。实现了所谓的Multi-Reactor模型，能够充分利用多核CPU实现以事件循环为基础的基本编程模型。同时在此基础上构建了Verticle这样类似Actor的概念，以应对并发编程的需求。

Vert.x的核心为EventBus和EventLoop，前者用户消息传输，作为联通各个Handler的神经系统；后者作为任务执行的调度者，保证高性能。任何使用Vert.x构建的应用，都必须围绕这二者作文章。否则就失去了使用它的意义。

## **核心类**

### Vertx

Vertx是最为核心的类，创建任何Vertx组件几乎都需要Vertx类的实例。

创建一个单机实例的方法是`Vertx.vertx()`，然后就可以使用了。以此为入口，我们看看Vertx在创建时都做了什么。

首先看

### EventBus



### Context



### Verticle



## **任务调度**



## **消息传输**



## **发布Verticle**



## **数据共享机制**



## **思维导图**



## **总结**