## 第五章 复制

复制主要讲的是分布式数据库通常所采用的复制算法。大致有如下三种，几乎所有分布式数据库都采用这三种

- 单领导者
- 多领导者
- 无领导者

复制的难点在于如何体现变更

### 领导者与追随者

一主多备，主库叫做领导者，备库叫做追随者。只有领导者才能接受写操作，备数据库只能接受读操作。主数据库按照一定的策略写入从数据库，达到数据的最终一致性

### 同步复制OR异步复制

同步：主数据库接收到写请求，除了写入本地数据库，还有写入从数据库，等从库响应写完后，再向客户端回复写完操作

异步：主库写入本地数据后，直接返回成功

半同步：一个从库采用同步策略，其他采用异步策略

### 数据库节点宕机怎么办

从库失效：从主库中恢复宕机后的部分数据即可

主库失效：比较麻烦，需要按照一定的策略再次确定新的主库，然后重新配置整个数据库系统。	

主库失效会有比较大的麻烦，有可能出现数据丢失、双主库等问题出现。

### 复制的方式

复制一般采用复制日志的方式实现，即主机按照一定的策略将变更以日志的形式发送给从机。日志的形式有几种

- 基于语句的复制：即直接在从机上应用更新的语句。但这样在自动生成字段上会造成数据不一致的情况
- 二进制日志：直接传输文件存储的日志格式。但这样没有可读性，且容易受到版本不一致所带来的问题
- 逻辑日志：不采用存储引擎的格式，而是将复制分离出来，进行逻辑复制。这样有可读性，且不会收到版本不一致带来的问题

### 复制延迟的问题及解决方式

之前所说的同步和异步复制的问题，异步复制会导致从库相对主库有一个延迟，只能保证数据的最终一致性。在延迟的这段期间，如果用户进行操作，会出现很多魔幻的情况。

#### 读自己写的内容

场景：主库写，然后才从机上读，因为延迟的原因，读不到自己刚才写的内容，感觉像是丢了一样

解决：解决方案就是读取时从已经修改过的库中读取，比如直接从主库读。

#### 单调读

场景：用户的两次读取，先读延迟小的从库，再读延迟大的从库，会先读到修改后的结果，再读到修改前的结果，好像时光倒流

解决：一个用户只能在同一个库中读取

### 一致前缀读

场景：一系列的写入按某个顺序发生，则任何人读取这些写入时，都要按照这个顺序出现

解决：确保具有顺序联系的数据都被写入在同一个分区或者分块中

### 多主复制

多主复制将数据库划分为多个区域，每个区域一个领导者，负责该区域的写入和复制分发。多个领导者的应用场景

#### 多数据中心

优点：

- 数据中心变多，用户可以在离自己比较近的数据中心写，对用户来讲性能变好了
- 数据中心可停机，不会影响整体的工作。对网络的容忍程度也更高

缺点：

- 两个数据中心可能会修改相同的数据，冲突的解决是个问题。
- 多主复制在很多数据库中都属于改装功能，总有些配置上的缺陷，因此使用它被认为是有缺陷的，应尽可能避免

#### 需离线操作的客户端

离线操作的每个客户端，都充当了其本地数据的领导者，每个设备都是一个数据中心。再次联网时，本地数据库再按照一定的策略与其它库进行同步

#### 协同编辑

允许多个人同时编辑文档。通常不会将这试做一个数据库复制问题，但是它与前面提到的离线编辑案例有很多类似。

#### 处理写入冲突

- 避免冲突：确保特定记录的写入都是通过同一领导者。这是比较推荐的做法
- 冲突合并：可以为每个写入做一个标记，允许一个写入覆盖另一个写入；自动合并这些写入；保留所有写入，并提示用户选择如何合并

- 自定义合并逻辑：部分数据库提供这样的选择

### 无主复制

无主复制即无领导的分布式实现，客户端直接将写入发送到几个副本中；或者有一个协调者节点代表用户进行写入，与领导者不同，该协调者不执行特定顺序的写入。

#### 节点故障时写入数据

客户端向多个副本写入数据，如果有其中的几个（这里的个数成为法定人数，可配）返回写入完成的数据，则认为写入成功。同样读取时会读取多个副本的数据，取其中版本最新的数据即可。这样，即使有节点宕机了，也不会影响整体的使用

- 读修复：当宕机节点恢复了，客户端读取到该节点数据版本过时，则将新版本数据写入，完成数据修复
- 反熵：数据库有一个后台进程，用于不断查找并解决不同数据版本之间的差异

## 第六章 分区

前面讨论的是数据在不同节点的副本的传输，称为复制。但是对于非常庞大的数据集或非常庞大的的吞吐量，仅仅进行复制是不够的。还需要进行分区，又称分片

分区在不同的数据库中有不同的叫法，有的叫分片、区域、表块等

分区的目标是将数据均匀地分布在各个区域上。

### 分区与复制

分区与复制通常是结合使用的，每个节点可以存储多个分区，节点之间的分区通过复制的方式同步数据，分区可以任意组合

### key-value数据的分区

分区的主要目标是提升数据量和吞吐量，将数据和查询负载均匀分布在各个节点上。如果足够平均，那么10个节点就能处理原有10倍的数据。

如果分配不均，称之为**偏斜**。极端情况下会偏移到一个节点。发生偏斜时很高负载的节点，称作**热点**

- 根据key的范围分区：缺点是某些情况下可能会出现较大偏斜，优点是范围查询性能较好
- 根据key的hash分区：优点是分配较为均匀，缺点是范围查询性能不好

#### 负载倾斜与消除热点

目前数据库并不支持自动检测和补偿倾斜，因此必须由应用程序自己做补偿。例如，如果一个key会在短时间内被大量访问，可以在该key上加随机数，分成更多个key，从而降低原热点的热度。使得系统不至于崩溃。

### 分区与次级索引

上面讲得都是依照主键进行分区，如果有次级索引呢？有两种次级索引对数据库进行分区的方法：基于文档的分区和基于关键词的分区

#### 基于文档的次级索引

每个分区独立，维护该分区自己在某个字段上的索引，查询时再将多个分区的次级索引结合进行查询。因此文档分区索引又称为本地索引。

#### 基于关键词的次级索引

构建一个覆盖所有分区的全局索引，每个节点都存储一个该索引，这样每次查询时就不用遍历分区了。

### 分区再平衡

原有分区后，数据库会发生各种变化，比如吞吐量需求增加、数据量大小增加等。难以避免地会将负载从集群中的一个节点向另一个节点移动。这个过程称为再平衡

再平衡应该满足如下要求：

- 平衡后负载均匀分配
- 平衡期间不影响数据正常访问
- 节点之间只移动必须的数据，以减少IO负载

#### 平衡策略

- 分区数量固定：一开始分配多于节点数量的分区，增加节点时，只需要将其它节点上的分区拿过来即可，简单高效。但是这种方式的分区数量在一开始就分配好了。不好把控。采用这种方式的如 Elasticsearch	
- 动态分区：当分区增长到超过配置的大小时，数据库自动将其分成两个分区，每个分区约占原有一半的数据。相反，每个分区的数据减少到一定阈值时，将会和相邻分区进行合并。这是按键的范围分区的数据库通常采用的策略，如HBase

- 按节点比例分区：使分区数与节点数成比例。当负载增加时，通过增加节点，从而增加分区，将原有节点的分区的数据移动一部分到新节点

### 请求路由

分布式数据库中，客户访问时如何知道自己应该访问哪个节点？——服务发现机制 (service discovery)，它不仅限于数据库，也适用于追求高可用性的一般软件。且一般有如下几种方案

1. 客户联系任何节点，该节点负责将请求转发到适当的节点

2. 将客户的所有请求转发到路由层，由路由层进行转发。

3. 要求客户了解节点及分区的分配，从而直接访问响应位置

   许多分布式数据库系统都依赖于一个独立的协调服务，如Zookeeper来跟踪集群元数据，每个节点都在其中注册自己，zookeeper维护分区到节点的可靠映射。其它参与者，如路由层订阅其中的信息。当分区改变时，Zookeeper就会通知订阅者，使其保持罪行的路由信息。

## 第七章 事务

### ACID

- 一致性：指的是数据库在应用程序的特定概念中处于良好状态。更好的表述是：对数据的一组特定陈述必须始终成立。例如，银行系统中，账户余额总是等于原有余额减去取出的钱。而这个特定陈述是由应用程序决定的，取决于应用程序，数据库只能通过原子性和隔离性来确保它。因此可以说C并不属于ACID

数据库的这些概念，因为商业化等因素，使得某些概念在不同数据库就具有不同的意义，概念上模糊了，这点注意。

### 单对象和多对象

- 单对象操作：数据库的自增操作、CAS操作、以及其它单对象操作被称为轻量级事务，而不应该被称为ACID

### 弱隔离级别

严格的ACID虽然能够保证数据的正确性，但是由于串行化带来的成本太高，很多数据库并不愿意承担性能损失，于是在隔离级别上进行一些弱化。

- 读已提交：即只能读到已提交的数据

  这是Oracle 11g， PostgreSQL，SQL Server 2012，MemSQL等数据库的默认设置

  其实现方式一般是在写上加行锁，一次只允许一个事务写入。读取上，数据库只暴露原有值和新提交值，不会暴露中间值

- 可重复读：即同一事务中，多次读取同一个值，能够得到一致的结果

  实现方式是快照隔离，即针对某个时间点对数据库进行快照，即使事务期间快照这部分内容被其它人修改了，也不会体现在该快照中。PostgreSQL，InnoDB引擎的MySQL，Oracle，SQL Server等都支持

  索引如何在快照隔离中使用？一种是对于新增加的内容也加入索引中，在查询时通过一定的规则过滤掉该新内容；二是使用写时拷贝的B树，为每个修改页面创建一个副本。

  快照隔离，可以认为是一个隔离级别，因为不同的数据库对它有不同的命名，Oracle中称为串行化，而MySQL和PostgreSQL称为可重复读。

### 防止丢失更新

当进行 读取 - 修改 - 写入数据库 操作时，可能发生丢失更新（自己曾经遇到过这个问题）。解决该问题的方式

- 原子写

  ```sql
  UPDATE counters SET value = value + 1 WHERE key = 'foo';
  ```

  上面的语句在大多数数据库中是并发安全的。类似的有的数据库还提供了自增等原子操作。原子操作通常通过在读取对象时，获取其上的排它锁来实现。以便更新完成之前没有其他事务可以读取它。

- 显式锁定

  使用select ... for update

- 自动检测丢失的更新

  事务管理器自动检测丢失的更新，并强制他们重试

- Compare And Set

  记录开始读取时的值，比较更新时的值与读取时的值，一致则更新，不一致则重试整个逻辑。

### 幻读

即一个事务的写入改变了另一个事务的搜索查询结果。幻读遵循如下模式

1. 使用SELECT查找出符合要求的行，病检查是否符合要求。
2. 按照上述检查结果，决定代码是否继续
3. 如果继续，则执行写入操作，提交事务。

当两个事务的第一步同时执行，第二部都得到相同的结果。一个先执行第三步，改变了另一个事务本来的先决条件，使得其不符合条件，从而错误地执行第三步，导致出现预期外的结果。

**解决**

物化冲突：幻读的原因，是操作多个对象，没有对象可以加锁。那我们可以手动给他加一把锁。比如会议室预定系统，可以创建每个会议室在每个时间段是否被预定的标记，针对该标记加锁。

物化冲突的缺点是使得数据库变得比较丑

### 可序列化

#### 真的串行执行

真串行执行会有很多限制，比如事务必须短小，否则会拖慢所有事务的处理。写入吞吐量必须低到能在单个CPU核上处理。

#### 两相锁定 （2PL）

这是几十年来广泛使用的序列化算法。其规定如下

- 如果事务A读取了一个对象，并且事务B想要写入该对象，那么B必须等到A提交或中止才能继续。 （这确保B不能在A底下意外地改变对象。）
- 如果事务A写入了一个对象，并且事务B想要读取该对象，则B必须等到A提交或中止才能继续。

2PL用于MySQL（InnoDB）和SQL Server中的可序列化隔离级别，以及DB2中的可重复读隔离级别

缺点：

两阶段锁定下的事务吞吐量与查询响应时间要比弱隔离级别下要差得多。这一部分是由于获取和释放所有这些锁的开销，但更重要的是由于并发性的降低。按照设计，如果两个并发事务试图做任何可能导致竞争条件的事情，那么必须等待另一个完成。

#### 乐观并发控制技术（序列化快照隔离）

2PL性能不好，真串行扩展性和性能都不好。而若隔离级别又有这样那样的问题。一个性能好又能满足串行隔离界别的算法很有前途：可序列化快照隔离。2008年首次被提出

序列化快照隔离是一种**乐观（optimistic）** 的并发控制技术。在这种情况下，乐观意味着，如果存在潜在的危险也不阻止事务，而是继续执行事务，希望一切都会好起来。当一个事务想要提交时，数据库检查是否有什么不好的事情发生（即隔离是否被违反）；如果是的话，事务将被中止，并且必须重试。只有可序列化的事务才被允许提交。

顾名思义，SSI基于快照隔离——也就是说，事务中的所有读取都是来自数据库的一致性快照。与早期的乐观并发控制技术相比这是主要的区别。在快照隔离的基础上，SSI添加了一种算法来检测写入之间的序列化冲突，并确定要中止哪些事务。

**原理**：

串行主要用于解决最终的幻读问题，前面说过其问题点，两个事务的决策都是基于过期的前提。SSI，就是为了解决这样的过时前提。为了提供可序列化的隔离级别，如果事务在过时的前提下执行操作，数据库必须能检测到这种情况，并中止事务。判断是否过期及是否中止成了关键。

判断是否过期通过如下两个方面考虑

- 检测对旧MVCC对象版本的读取（读之前存在未提交的写入）
- 检测影响先前读取的写入（读之后发生写入）

**性能**：

SSI的性能取决于中止的次数。相对于MVVC，性能损失并不是特别大。



## 第八章 分布式系统的麻烦

本章主要介绍了分布式系统中通常会遇到的问题，以及通常的解决方式。

我们可以假设分布式系统中，总会出现这样那样的问题，节点那么多，每个时间总会有几个不能工作，网络延迟等等。但分布式系统的目标就是基于不可靠的组件构建出可靠的系统。

### 不可靠的网络

网络本身是不可靠的，及时网络可靠，也可能因为目标节点忙于做其它事情而产生延迟，而延迟多久可以判定一个节点挂了，这也是不确定的。

可以通过各种方式改变延迟对本身系统的影响，这是一个成本/收益的衡量结果

### 不可靠的时钟

时钟在分布式系统中也是很重要，各个节点之间如果时间不同步会让很多内容产生误会，例如无法弄清数据的发送顺序。

#### 单调钟与时钟

时钟：即我们常说的时钟，如Java.currentTimeMills()返回的是UTC时间，这种时间可以被调整，即如果与标准时间偏差太多，它是可以被调整回去使得偏差较小的。由于其可调整性，因此用来测量经过时间时就不是很好

单调钟：适用于测量经过时间，如Java.nanoTime()，它是一直向前走的，不会被调整。

#### 依赖同步时钟

分布式系统中依赖同步时钟是很危险的。例如，在前面冲突解决上有一种最后写入为准的策略，这里判断谁先谁后，如果依据时间，各个节点的时间可能有偏差，从而造成谁先谁后判断错误，则冲突解决会错误。这会导致一些数据丢失。

### 知识、真相与谎言

#### 真理在多数人手中

单个节点很可能是不可靠的，可能随时失效，这样会使得整个系统卡死。一般的分布式算法依赖于法定人数，即在节点之间进行投票，决策需要来自于多个节点的最小投票数。

例子：一主多从的结构中，如果领导者卡死，被法定人数投票认为死掉了，选出了新的领导者。那么当原领导者复活时，由于法定人数不再承认它是领导者，从而不会造成脑裂的情况（即有两个领导者）

#### 拜占庭故障

假设节点会说谎，在不信任的环境中达成共识的问题称为拜占庭将军问题

当一个系统在部分节点发生故障、不遵守协议、甚至恶意攻击、扰乱网络时仍然能继续正确工作，称之为**拜占庭容错（Byzantine fault-tolerant）**的，在特定场景下，这种担忧在是有意义的

这类问题防治起来非常复杂，一般采用传统机制：认证、访问控制、加密、防火墙等

#### 系统模型

分布式算法，关于定时的假设，有如下三种模型：

- 同步模型：算法假设网络延迟、进程暂停、时钟误差都是有界限的。
- 部分同步模型：算法假设大部分情况能够想同步模型一样运行，但有时会超出网络延迟。这是很多系统的现实模型
- 异步模型：算法不允许为同步做任何假设，它甚至没有时钟

关于节点失效，常见的节点系统模型：

- 崩溃-停止故障：算法假设节点只能通过崩溃失效，意味着节点可能在任意时刻突然永远地消失
- 崩溃-恢复故障：算法假设节点可能会在任意时刻崩溃，但也许会在位置时间重新开始响应。
- 拜占庭故障：节点可以做任何事，包括发送虚假消息给其它节点

安全性与活性

- 安全性：没有坏事发生

- 活性：最终好事发生

  区分安全性和活性属性的一个优点是可以帮助我们处理困难的系统模型。对于分布式算法，在系统模型的所有可能情况下，要求始终保持安全属性是常见的。也就是说，即使所有节点崩溃，或者整个网络出现故障，算法仍然必须确保它不会返回错误的结果（即保证安全性得到满足）。

  ​	但是，对于活性属性，我们可以提出一些注意事项：例如，只有在大多数节点没有崩溃的情况下，只有当网络最终从中断中恢复时，我们才可以说请求需要接收响应。部分同步模型的定义要求系统最终返回到同步状态——即任何网络中断的时间段只会持续一段有限的时间，然后进行修复。

