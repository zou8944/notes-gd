# Zeko是如何做的

## 核心类

**ZekoVerticle**

一个抽象类，定义了如下几类公共方法

- bindRoutes，使用RouteSchema为route绑定更多路径，用户
- handleRuntimeError，定义公共的处理错误的方案

**ApiController**

一个抽象类，用户的Controller都继承它，它主要定义了validateInput方法，用于做输入验证，以及定义一些快捷的响应函数。

**各种注解**

用于路由的注解：@Get、@GetSuspend

用于做参数验证和Open API生成的注解：@Params

这两类注解可以解决手动路由定义、参数验证、Open API生成三个核心问题。也是该库主要解决的内容。

## 代码生成原理

继承AbstractProcessor，即APT，生成了如下三种内容

- 根据controller的Params注解生成验证规则
- 根据controller上的@Get等注解生成路由Route
- 根据上述两者生成swagger的描述文件

## zeko-example中，各层如何工作

使用传统的MVC分层，通过依赖注入工具koin将各层进行粘合。

### Controller

继承ApiController

### Service

就是一个非常普通的类，主要集中了逻辑

### DAO

也是一个非常普通的类，包含了zeko自己的DB框架写的数据库代码

## 使用到了什么设计模式

### 建造者模式

CircuitBreakerBuilder

提供一个make方法，根据输入的内容，创建对应的断路器。主要是传入了断路器参数和名字。

## 值得学习的点

- utilities是通过扩展方法的方式实现的，文件名小写。

- 支持协程的方式：Service类直接写suspend方法，在Verticle中调用，Verticle继承了CoroutineVersicle
- 项目简洁易懂，Java代码功力比较好
- 是在Vertx上实现MVC分层的比较好的示例

## 项目缺点

- 用Java的思路写kotlin，充斥着大量的伴生对象的方法
- 没有注释
- 该项目既没有展示Vertx的特点，也没有充分展示Kotlin的特点，但看起来又比较简洁。可以说有一点不伦不类。

## 项目总结

该项目的核心仅在Vertx的Web库，通篇围绕着Vertx-Web做文章，具体来讲，它做了如下几点

- 抽象出Controller层，用@Get之类的注解代替显式的设置路由；用@Param注解代替Open API描述文件的定义
- 代码生成：如果说抽象Controller是逻辑上的核心，代码生成则是实现上的核心，完全手撸Route绑定类、参数验证类、Open API描述文件的生成代码。

通过以上，zeko解决了三个Vertx Web编程的痛点：路由定义、参数验证、swagger描述文件定义。

但看起来有点刚，需要手撸的内容实在过多。生撸啊！！！

# Zeko-SQL-Builder是如何做的

**如何查询数据**

自己抽象了一系列类，核心是Query类，该类包含了属性（代表SQL的各个不同的部分，如select的字段值，from的表名等），再通过table()，from()等方法设置对应的属性值。最后调用compile将各部分的字段组合起来，得到最终的SQL。

优点：实现简单，原理简单，相对于直接写SQL来说，规避了裸写SQL的语法错误

缺点：类型不安全，无法验证各字段类型

**如何执行SQL**

使用DBSession封装了底层client的实现方式，只暴露SQL、参数、结果，其中，结果是被处理成Map的

# Zeko-Data-Mapper是如何做的

核心类是DataMapper和MapperConfig

**DataMapper**

定义了mapStruct()方法，接受表格字段的映射规则，和SQL-Builder查询的结果。得到根据映射规则的最终处理结果。

**MapperConfig**

定义了表格的字段映射规则。

# Koin做依赖注入的原理是什么

