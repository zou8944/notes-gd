K8S整理

1. 目前线上的pod有下面这么多

   ```bash
   NAME                                 CPU(cores)   MEMORY(bytes)   
   coordinator-5c47f65ccb-cgtmw         0m           100Mi           
   ergedd-5c48dfb4cf-dv4l8              17m          391Mi           
   ergedd-schedulers-8f8665ffc-2l5df    0m           224Mi           
   hula-admin-74dfcc546c-5pskc          0m           219Mi           
   hula-slate2html-5fd979b999-bt7p5     0m           38Mi            
   hula-ttp-54f9496-9k2wz               11m          358Mi           
   hula-ttp-bot-9fdfc677c-7d7vv         2m           149Mi           
   monitor-grafana-0                    0m           21Mi            
   monitor-influxdb-0                   10m          415Mi           
   outlets-9dc8df55b-x8srf              0m           5Mi             
   postgres-0                           2m           29Mi            
   recommend-service-5c96699ff-n27zs    0m           270Mi           
   redis-ha-default-server-0            2m           2Mi             
   redis-ha-default-server-1            3m           2Mi             
   redis-ha-default-server-2            3m           2Mi             
   test-gptvg                           4m           1487Mi          
   tuner-fc67c889c-sql44                1m           142Mi           
   user-scala-ergedd-5dff7857b6-4k85q   35m          376Mi           
   user-server-v2-865cb74946-rvd8w      6m           341Mi
   ```

2. 各个应用的重要程度和限制

   

- 这里推荐，ECS节点的CPU：MEMORY比，在Java这类应用为1:8

- 这里说得貌似是创建一个伸缩组，指定已创建的ECS节点在其中运行

  https://help.aliyun.com/document_detail/119099.html?spm=a2c4g.11186623.6.969.170f4e99Y8U3JN

  - 需要创建

- 



## 文档输出要点记录

- ECS和ECI价格对比，怎么组合比较划算

- 要不要使用ECI、要不要使用Serverless Kubernetes?

  ECI可以用来应对突发状况；Serverless Kubernetes遵从阿里云建议，只用在批量任务、CI/CD等情况（话说：我们是不是可以建一个Serverless集群，用来做CI，以提升编译速度）

  但主要还是使用ECS。

- 发布怎么搞？

  新应用发布可以用helm.

- 监控怎么搞？


![image-20201122175237437](/home/floyd/.config/Typora/typora-user-images/image-20201122175237437.png)

每个服务整理的点

- cpu、memory怎么配置的？
- 用到了多少磁盘？哪里的磁盘？费用如何？
- 亲和性怎么配置的？

## 服务

### 集群内客户端访问集群内的服务

1. 默认创建的服务仅在集群内部有效，只能在集群内部访问，其服务于多个pod时，是随机选择一个pod。**注意是随机哟**

2. 默认创建的服务为什么ClusterIP不一样

   因为这个集群IP不是集群的IP的意思，而是创建的服务在集群内部的IP

3. 服务的会话亲和性

   每次将同一个客户端的请求都指向同一个pod

   ```yaml
   apiVersion: v1
   kind: Service
   spec:
     sessionAffinity: ClientIP
   ```

   会话亲和性仅支持None和ClientIP两种。

   **那么它是如何判断请求是来自同一个客户端的呢？**

4. 如何做服务发现

   主要描述如何在集群内部在不知道IP的情况下找到目标服务。

   - 通过环境变量

     新创建的pod，会将集群中已有的服务写到新pod的环境变量中。

     缺点：如果需要的服务在目标pod之后创建，那目标pod的环境变量将不会有该服务

   - DNS

     集群中有一个叫kube-dns的pod，用来管理集群内部的dns服务

     pod可以通过全限定域名（FQDN: Full Qualify Domain Name）访问到目标服务。格式如下

     ```bash
     <sevice-name>.<namespace>.<suffix>
     hula-ttp-svc.default.svc.cluster.local
     ```

     如果服务和使用它的pod在同一个命名空间内，甚至可以省略命名空间和后缀，直接通过服务名访问到该服务。这也是我们现在使用的方式。

     **注意：pod也可以不使用集群内的dns服务，由配置项dnsPolicy属性决定**

5. 服务的IP由于是虚拟IP，因此事ping不通的。

### 集群内的客户端访问集群外的服务

指的是通过service访问集群外的地址。比如访问www.baidu.com

首先了解endpoint

1. endpoint是一种资源，就是暴露一个服务的IP地址和端口的列表

   ```bash
   kubectl get endpointszz
   ```

2. 这意味着它和服务是绑定的

   ```bash
   floyd@floyd-ThinkPad-T490:~$ kubectl describe svc user-server-v2-svc
   Name:              user-server-v2-svc
   Namespace:         default
   Labels:            <none>
   Annotations:       <none>
   Selector:          app=user-server-v2
   Type:              ClusterIP
   IP:                172.21.4.96
   Port:              grpc  12345/TCP
   TargetPort:        12345/TCP
   Endpoints:         172.20.0.136:12345	# 这里
   Session Affinity:  None
   Events:            <none>
   ```

3. 服务yaml文件中指定了匹配的pod，但在连接重定向时却不会直接使用它。它只是用来创建endpoint资源的，真正起作用的是endpoint资源。连接进来时，服务从他们中选择一个进行连接。

4. 我们也可以手动配置服务的endpoint，实现通过服务访问任意地址

   - 创建不包含任何pod的服务
   - 创建endpoint资源，指明要将服务重定向到的目标ip和端口

   这样就实现了external-service，通过服务访问到了集群外的资源。

5. 除了endpoint，还可以创建ExternalName类型的服务来访问外部服务、

   ```yaml
   kind: Service
   metadata:
     name: external-service
   spec:
     type: ExternalName
     externalName: xxx.xxx.com
     ports:
     - port: 80
   ```

   注意：ExternalName是在kube-dns中创建了CNAME类型的记录，而不是ip类型。

### 集群外的客户端访问集群内的服务

#### 方式1 - NodePort

创建NodePort类型的服务：在每个节点上暴露一个端口，访问该节点端口时会将流量转发给服务，服务在转发给它的Pod。创建它时需要指定三个内容

- 该服务在集群IP的端口号
- 目标Pod的端口号
- 节点的端口号

创建后，服务会得到一个集群IP，而它对应的External-IP为\<nodes>，即节点。

因此可以内网通过两种方式访问到该服务

- 集群IP:端口 - 该请求会被转发到节点指定端口
- 节点IP:端口 - 该请求会被转发的对应的服务

但是外网就只能通过如下方式访问服务。因为集群IP是内网IP，节点IP才是外部IP。

- 节点IP:端口 - 该请求会被转发的对应的服务

注意：以NodePort的方式必须在K8s云服务提供商的节点安全防火墙开启该端口，才能被正常访问。

#### 方式2 - 负载均衡

负载均衡是NodePort的扩展，比NodePort多的是在节点和客户端之间的SLB。SLB是云服务商提供的一个具有独立IP的资源，客户端可以直接访问它，它以负载均衡的方式将流量分发给其背后的节点。

1. 前面说过NodePort服务，请求从一个节点进入，再进入服务，再进入Pod，不一定进的是该节点的Pod。这会造成不必要的网络跳数。解决方式是

   ```yaml
   spec：
     externalTrafficPolicy: Local
   ```

2. 正常情况下，SLB是对所有Pod的负载均衡，但使用上述注解后，SLB不是针对Pod的负载均衡，而是对节点的负载均衡。如果两个节点有三个Pod，则一个Pod会被分到50%的流量，另外两个会被分到25%的流量

3. 从节点端口进来的请求包，很可能被执行了SNAT，从而导致在Pod无法拿到客户端的源IP。

#### 方式3 - Ingress

上面都只介绍了暴露一个服务的方式，对SLB，每个服务都要有一个IP，访问起来甚是麻烦。需要一个统一IP，该IP能访问多个服务。类型Nginx，反向代理的作用。

1. Ingress必须要Ingress控制器的存在才能正常工作

2. Ingress工作在HTTP层，不同于上面介绍的服务，它们工作在TCP层

3. Ingress工作原理（书上的方式）

   1. 客户端访问域名，通过DNS得到Ingress控制器的IP
   2. 客户端访问控制器IP，带上Host:<目标域名>请求头
   3. 控制器根据请求头的Host域名，向Ingress服务中查询其对应的服务，在获取服务的所有endpoint，即pod地址列表
   4. Ingress从这些endpoint选择一个pod，然后将请求转发过去

   ![](https://gdz.oss-cn-shenzhen.aliyuncs.com/notesimage-20201122110744687.png)

4. Ingress工作原理（阿里云方式）

   1. 客户端访问域名，通过DNS得到负载均衡的IP
   2. 客户端访问SLB，带上Host:<目标域名>请求头
   3. SLB将请求均衡地转发给两个Ingress控制器
   4. 后面的步骤和上面就一样了

   ![image-20201122111727412](../../.config/Typora/typora-user-images/image-20201122111727412.png)

5. Ingress的TLS传输。

   客户端到Pod之间的整个传输路径，只有客户端到Ingress控制器之间是TLS安全传输的，Ingress到Pod之间不是。

   这个好理解，就像是Nginx一样。

   要让Ingress支持TLS，只需要添加tls配置项即可。

### pod准备好了吗

通过标签匹配pod，那什么时候pod能够被接入服务呢？——就绪探针。

- 只有就绪探针运行成功时，该pod才会被加入服务
- 就绪探针会定期被服务运行，如果失败，pod会被从服务中移除

#### 就绪探针和存活探针的区别

- 就绪探针在检测失败后，并不会终止或重启pod，只是会将其标记为不可用（例如从服务中移除）
- 存活探针在检测失败后，会以一定策略终止或重启pod

### headless服务

**背景**

常规服务只是暴露一个集群内的IP，然后通过IP或FQDN访问服务，由服务去负责pod访问。如果我们想要自己决定访问哪个pod，即想要获取所有pod的地址，此时就需要headless服务

**实现**

只需将ClusterIP显式设置为none即可构建无头服务。通过dns查找服务的FQDN，即可得到所有pod的地址列表

**注意**

headless服务也可以直接通过FQDN访问，它会轮询所有pod，这也达到了负载均衡的效果

### 排查服务不通的方法

1. describe服务，查看endpoint是否有内容
2. 如果endpoint没有东西，很可能是pod没有正常启动
3. 如果endpoint有东西，集群内单独访问pod确定pod是否预期工作
4. 如果pod单独访问不通，说明还是pod的问题
5. 如果pod能访问通，可以telnet <服务ip> <端口>，如果不通，说明服务设置有问题，提工单问吧。

## StatefulSet

### 什么是有状态应用

一个典型的有状态应用，分布式数据库，有如下特点

- 每个pod有明确稳定的网络标示：每个pod都是从0开始的顺序索引。扩容时索引顺序增加，缩容时优先删除高索引值的pod。

- 每个明确的pod都有专属稳定的存储：这些存储并不会随pod的删除而删除。当因为缩容删除了pod，再扩容时，原来的数据卷会挂在扩出来的pod上。

- StatefulSet会保证pod的at-most-one语义，即除非非常明确地确定pod已经被删除了，它绝不会再创建一个pod来替代它（如果贸然创建一个新的，会使得可能两个pod对应一个存储卷，这很危险）

  因此，当pod长期处于不明状态时，StatefulSet永远不会删除它，此时需要我们手动删除。

### 访问有状态应用

无状态应用的服务会从pod中随机选取一个，因为它是无状态的，所以这很正常。

有状态应用则不行，每个pod都是独一无二的。因此需要创建一个headless Service。这样设置后有两种方式访问到pod

1. 域名：\<podName>.\<serviceName>.\<namespace>.svc.cluster.local

   比如：a-0.foo.default.svc.cluster.local

   不过这种情况是知道pod编号时才能这样访问，那如何通过Service的FQDN得知其下pod的FQDN呢？答案是SRV记录。

2. SRV记录

   SRV记录，是DNS记录的一种。又称作服务定位器，用于指明一个域名后面提供服务的机器(域名)。向ServiceFQDN发起一次SRV记录查询，即可得到其下的pod的FQDN。

### 疑问：第五章直接nslookup和这里的dig SRV有什么区别

- nslookup：能够查出服务域名后提供服务的pod的ip列表，但无法区分那个ip是哪个pod，且随着pod的伸缩，ip会变
- dig SRV：获取的是服务下pod的FQDN，它是稳定的，可以一直使用。

## 资源调度

1. 调度器只关注资源requests，即请求了多少资源，不关注实际使用量。如果一个单核节点，应用A请求了800m核，但是只是用了200m核，那新的应用即使只请求300m核也无法被分配到这个节点。

2. 内存调度的逻辑也和CPU一致。

3. 根据资源匹配节点有两种策略：优先匹配请求量最少的节点 和 有限匹配请求量最多的节点

4. kube-system命名空间内有一些系统组件会吃掉一些节点资源

5. CPU的requests如何影响CPU分配，假设有一个单核CPU

   - 如果pod A请求100m核，pod B请求了500m核，剩余400m核。如果两个CPU都尽情地跑，则pod A能够分配到剩余CPU的1/6，pod B能够分配到剩余CPU的5/6
   - 如果任何一个pod都没有跑超过请求的CPU，则剩余的400m核可以被另一个pod全部占满
   - 在第二种情况下，空闲的pod突然跑满的话，原先占满剩余CPU的容器会被限制回去，最终达到第一种情况的状态。

6. CPU这个资源可以无痛地在节点之间转移。但内存不行，如果某个故障pod或恶意pod占满了所有内存。则其它pod将无法使用。

7. 由于上面这一点，因此pod的内存limits一定要设置。

8. 创建pod时，只设置limits不设置requests的情况下，requests默认被设置成和limits一样的值。

9. limits可以超卖，即所有pod的limits之和超过100%。**需要注意的是，如果实际中节点使用量超过了100%，一些pod会被杀掉。**

10. **单个容器尝试使用比自己指定的limits更多的资源时也可能会被杀掉。这叫做OOMKilled(Out of memory killed)**

    要查看一个容器是否被OOMKilled，只需要kubectl describe pod xxx，查看其中的State和Last State两项即可。

11. 在容器内是怎么看这些限制的？

    在容器中并看不到这些限制。在容器中执行top，会看到它还是以节点的整体资源计算百分比。

    这样会导致两个问题

    - 内存上，应用在根据环境的百分比申请内存资源时，申请结果会超过设置的limits值，导致直接被OOMKilled
    - CPU上，如果应用根据当前系统检测到的核心数启动对应的线程，可能启动非常多线程，造成频繁的线程切换

12. 关于QoS等级

    当实际使用资源超过节点总资源时，有容器会被驱逐，驱逐的优先级有三个，从上到下优先级一次升高

    - BestEffort 不配置requests和limits
    - Burstable  不满足Guaranteed的都是Bustable
    - Guaranteed 设置的requests和limits的必须相等，且CPU和内存都要设置，所有容器都要这样设置（pod内允许多个容器）

    其中Guaranteed只有在系统进程需要内存时才会被杀掉。而BestEffort会最先被杀掉，Burstable其次。

13. 两个同等级QoS的进程谁先被杀掉？

    取决于进程的OOM分数：实际占用内存量/申请内存量。分数高的优先被杀掉。

14. 为了解决必须在每个pod都执行requests和limits的麻烦事，可以在一个命名空间下创建一个LimitRange，指定该命名空间下的pod或容器的默认requests和limits。

15. LimitRange只对单独的pod有限制作用，但不能解决上面所说的超额驱逐容器的问题。

    针对这一点，可以设置命名空间的限额，规定该空间内总共能够用多少资源。这个资源叫做ResoueceQuota

    ResourceQuota不仅可以限制CPU和内存的requests和limits，还可以限制PVC、POD、RC、Service等资源的数量。

### 获取实际资源使用情况

1. 每个节点都会包含一个名为cAdvisor的agent，他会收集整个节点和节点上运行的所有单独容器的资源消耗情况。

2. 而将所有节点的cAdvisor信息收集起来的，是一个叫做heapster的pod，它通过service暴露，使得外部可以通过一个稳定的IP地址访问它。

3. 使用kubectl top pod查看pod的实际使用情况

   ```bash
   kubectl top pod
   kubectl top pod --all-namespace
   kubectl top pod --container
   ```

4. 不过heapster有一个问题：它们只会保存一段时间的pod资源使用情况。必须使用额外的工具进行长时间监控。

5. 我们目前没有启动prometheus，因为占用资源太多。

## 自动伸缩pod与集群节点

k8s中可伸缩的内容

- pod的横向和纵向伸缩
- 节点的横向伸缩

### pod横向自动伸缩

1. pod横向自动伸缩由Horizontal控制器执行，我们通过HorizontalpodAutoscaler(HPA)来启动和配置Horizontal控制器。
2. 自动横向伸缩分为如下三个步骤

- - 获取被管理资源所管理的所有pod度量
  - 计算度量的平均数值是否达到预先设定的平均值
  - 超过平均值就增加pod，进行横向伸缩

1. pod度量也是从heapster拉取的
2. 需要pod的数量计算方式

所有pod的度量之和/预先设置的度量=副本数

假设现有三个pod，pod的cpu占用率依次是20%，30%，50%，希望每个pod保持的占用率为30%，则需要的pod数量为4个。

预先设置的值是再HPA中指定的。

1. 度量的计算方式：实际占用资源的量/请求的资源量。如实际占用100m核，requests设置为200m核，则得到的度量为50%。
2. 能够被自动伸缩的资源包括

- - Deployment
  - ReplicaSet
  - ReplicationController
  - StatefulSet

1. 能够基于哪些度量进行自动伸缩

- - CPU占用率（可以无痛伸缩）
  - 内存占用率（由于内存需要应用主动释放，k8s不方便管理，因此不适合用来做自动伸缩依据）
  - QPS
  - 其它，甚至可以基于ingress的流量进行伸缩

### pod纵向自动伸缩

k8s纵向自动伸缩再2018年还没弄好，不过目前好像有了。有空可以看看阿里云文档

### 节点自动扩容

1. 节点自动扩容由Cluster Autoscaler控制，**云服务中应该是有，有空研究一下是否可以配置阿里云的动态节点伸缩？在pod资源占满时申请新的临时节点，并发消息通知我们，到后面再去处理它。**
2. 涉及到申请新节点和归还节点，这个需要和具体云厂商查看。

kubectl get hpa, deployment # 依次列举多个类型的资源

Deployment是什么？

答：使用ReplicaSet只能控制pod的创建，而要滚动升级时，会新创建一个ReplicaController，拉取新版本的镜像，这个步骤很麻烦。于是用Deployment将其封装起来（当然实际上并不是这么做的，这个具体可以看看第11章）

如何获取一个pod的QPS？

是否可以配置阿里云的动态节点伸缩？在pod资源占满时申请新的临时节点，并发消息通知我们，到后面再去处理它。

记录点

- 每个pod的实际占用量超过一定限额时，应该需要向我们报警
- 当有pod被驱逐时，应该向我们报警


## 高级调度

### 污点

**什么是污点**

污点是节点的一种属性，和label、annotation一个级别。结合pod配置的污点容忍度，可以决定一个pod是否被分配到某个节点。

污点的格式是：key=value:effect ，value可以不写，effect是污点的效果。

污点总共有如下几种效果

- NoSchedule

  如果pod没有容忍该污点，则不能被调度到该节点

  只影响pod调度，不影响已经在运行的pod

- PreferNoSchedule

  如果pod没有容忍该污点，则最好不要调度到该节点。如果实在没有其它节点了，也可以调用过来。

  只影响pod调度，不影响已经在运行的pod

- NoExecute

  调度期间的作用同NoSchedule，此外它还影响pod执行。如果节点新增了一个该效果的污点，则原先运行在该节点上不能容忍该污点的pod会被移除。

**查看节点污点**

kubectl describe node xxx

其中的Taints属性

**查看pod污点容忍度**

kubectl describe pod xxx

其中的Tolerations属性

**添加污点**

kubectl taint nod xxx key-value:NoSchedule

**设置节点失效后pod重新调度的等待时间**

容忍度也可以配置等待时间，表示如果已经运行在节点上的pod，如果节点不再满足容忍度，等待多久再重新调度。

这种情况可以使节点短暂下线。

### 节点亲和性

以前有nodeSelector可以用，但是它的功能太狭窄。于是有了节点亲和性，它有更加复杂的表现形式。

节点亲和性类型

- requireDuringSchedulingIgnoredDuringExecution

  仅在调度期间有效，且调度期间必须满足该条件

- preferredDuringSchedulingIgnoredDuringExecution

  仅在调度期间有效，且调度期间尽量满足该条件

**优先级节点亲和性规则**

可以指定多个规则，每个规则占比不同。比如指定了preferredDuringSchedulingIgnoredDuringExecution的亲和类型，执行匹配app:v1标签优先级为80%，app:v2标签的优先级为20%。则会被分配的节点优先级如下

- 最高优先级：同时又app:v1和app:v2的节点
- 次1优先级：有app:v1的节点
- 次2优先级：有app:v2的节点
- 次3优先级：其它节点

**题外话**

1. node有三个固定的标签，用来反映节点所在区域和主机地址等
   - failure-domain.beta.kubernetes.io/region=europe-west1
   - failure-domain.beta.kubernetes.io/zone=europe-west1-d
   - kubernetes.io/hostname=xxxxx

### pod亲和性和反亲和性

配置和节点也很类似，不同的是需要指定topologyKey

**topologyKey**

topologyKey就是匹配到的pod所属节点的标签。意义是，找到匹配到的pod，查看其对应节点的key为topologyKey的标签值，将本pod调度到具有该标签的的节点上。因此常看到topologyKey设置为kubernetes.io/hostname。

由于topologyKey的值也只是node上的标签key，因此利用不同level的节点标签，我们可以设置一些pod分配在同一主机、同一机柜、同一区域、同一国家等。

**被亲和**

如果pod1设置了对pod2的亲和性，两个pod被分配到同一个node。此时将pod2删除，则新的pod2还是会被调度到原来的node，即使pod2并没有设置对pod1的亲和性。

**反亲和**

亲和的反，也好理解。

**题外话**

1. 书里面有调度器日志，是从哪里打印出来的？

## 额外

### fabric8

试用一下fabric8这个东西：http://fabric8.io/guide/overview.html






## 问题

1. 命名空间应该如何规划使用，才能达到最好的效果？



## 用到的命令行

```bash
kubectl describe nodes # 可以看节点状态
kubectl describe quota # 查看当前命名空间配额设置
kubectl cluster-info         # 查看集群信息，包含集群地址、metrics地址、dns等

kubectl exec <podName> -- curl -s htt[s://wwegawg # 在某个pod内部执行一段命令。--表示后面开始是pod内部的命令

kubectl proxy    # 在本地起一个代理，代理到API Server上
```



