---
title: Kubernetes的设计解读
date: 2022-03-20 12:31:00
tags:
  - Kubernetes
categories: Kubernetes
---
### pod 设计解读

在kubernetes中，创建、调度、管理的最小单位是pod

- pod是IP等网络资源的分配的基本单位，这个IP及其对应的network namespace是由pod里面的容器共享的
- pod内的所有容器页共享volume。当一个volume被挂载在同属同一个pod的多个Docker容器的文件系统
- IPC namespace 即同一个pod内的应用容器能够使用System V IPC或者POSIX消息队列进行通信
- UTS namespace 即同一个pod内的应用容器共享主机名

1.label 和label selector与pod协作

2.pod的现状和未来

- 资源共享和通信
- 集中式管理，指pod内的所有容器资源

3.pod内的容器网络与通信

​	通过pause容器进行pod内的容器网络与通信

- replication controller设计解读

  replication controller在设计上依然体现出了"旁路控制"的思想，为每个pod “外挂”了一个控制器进程，从而避免了健康检查组件成为性能瓶颈

  replication controller只能与重启策略为Always的pod进行协作

  replication controller的经典场景:

    - 重调度
    - 弹性伸缩
    - 滚动更新
    - 多版本应用release追踪

- service的设计解读

  service通过标签label将流量负载均衡到对应label标签的pod上

    - service工作原理

      Kubernetes集群上的每个节点都运行着一个服务代理(service proxy),它是负责实现service的主要组件

      kube proxy两种工作模式：

        - userspace模式

          对于每个service kube-proxy都会在宿主机监听一个端口与这个service对应起来，并在宿主机上建立起iptables规则，将service IP:service port的流量重定向到上述端口

        - iptables模式

          iptables模式下的kube-proxy将负责创建和维护iptables的路由规则，其余工作交由内核态的iptables完成

    - service的自发现机制

        - 环境变量方式
        - DNS方式(mysvc.myns)

    - service 外部可路由性设计

        - NodePort
        - LoadBalancer
        - external ip

- 新一代版本控制器 replica set

  replica set 用于保证label selector 匹配的pod数量维持在期望状态

  replicat set 与 replication controller 的区别是：replication controller只支持等值匹配   replicat set支持基于子集的查询

- Deployment

  Deployment多用于pod 和replica set 的更新，可以方便地跟踪观察其所属的relica set或者pod的数量以及状态变化

- DaemonSet

- ConfigMap
- Job

### Kubernetes核心 组件解读

#### Master节点：

##### APIServer:

Kubernetes APIserver负责对外提供Kubernetes API服务，它运行在Kubernetes的管理节点-Master节点

* APIServer的职能:

    * 对外提供RESTful的管理接口
    * 配置Kubernetes的资源对象
    * 提供可定制的功能性插件

* APIServer启动过程:

  1.新建APIServer 定义一个APIServer所需的关键资源

  2.接受用户命令行输入，为上述各参数赋值

  3.解析并格式化用户传入的参数

  4.初始化log配置

  5.启动运行一个全新的APIServer

* APIServer对etcd的封装：

  Kubernetes使用etcd作为后台存储解决方案，而APIServer则基于etcd实现了一套RESTful API，用于操作存储在etcd中的Kubernetes对象实例

* APIServer如何保证API操作的原子性:

  Kubernetes的资源对象都设置了resourceVersion作为其元数据的一部分，APIServer以此保证资源对象操作的原子性

##### Scheduler:

根据特定的调度算法将pod调度到指定的工作节点上，这一过程称作绑定

* Scheduler的数据采集模型

  Scheduler定时向APIServer获取各种各样它需要的数据，为了减少轮询时APIServer带来的额外开销，对于感兴趣的资源设置了本地缓存机制

* Scheduler调度算法

  Kubernetes中的调度策略分为两个阶段：Predicates , Priorites

    * Predicates :回答能不能
    * Priorites：在Predicates基础上回答匹配度

* controller manager

  kubernetes controller manager 运行在集群的master节点上，管理着集群中的各种控制器

#### 工作节点：

##### cAdvisor:

获取当前工作节点的宿主机信息

##### kubelet :

kubelet组件 是Kubenetes集群工作节点上最重要的组件进程，它负责管理和维护在这台主机上运行着的所有容器。本质上它的工作可以归纳为使得pod的运行状态（status）与它的期待值（spec）一致。

Kubelet如何同步工作节点状态：

1.kubelet调用APIServer API向etcd获取包含当前工作节点状态信息的node 对象，查询的键值就是kubelet所在工作节点的主机名，

2.调用cAdvisor客户端API获取当前工作节点的宿主机信息，更新前面步骤获取到的node对象

3.kubelet再次调用APIServer API将上述更新持久化到etcd中

##### kube-proxy :

Kubernetes基于service、endpoint等概念为用户提供了一种服务发现和反向代理服务，而kube-proxy正是这种服务的底层实现机制。

服务发现实现：

Kube-proxy使用etcd的watch机制，监控集群中service和endpoint对象数据的动态变化，并且维护一个从service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响。

kube-proxy主要有两种工作模式: userspace 和 iptables

userspace模式：

iptables模式：

iptables模式下的proxier值负责在发现变更时更新iptables规则，而不再为每个service打开一个本地端口，所有流量转发到pod的工作将交由内核态的iptables完成。

### 核心组件协作流程：

#### 创建pod

当客户端发起一个创建pod的请求后，kubectl向APIServer的/pods端点发送一个HTTP POST请求，请求的内容即客户端提供的pod资源配置文件



APIServer收到该REST API请求后会进行一系列的验证操作，包括用户认证，授权和资源配额控制等。验证通过后，APIServer调用etcd的存储接口在后台数据库中创建一个pod对象。



Scheduler使用APIServer 的API  定期从etcd获取或者监控系统中可用的工作节点列表和待调度pod , 并使用调度策略为pod选择一个运行的工作节点，这个过程叫绑定



绑定完成后，scheduler会调用APIServer的API在etcd中创建bingding对象，描述在一个工作节点上绑定运行的所有pod信息。同时kubelet会监听APIServer上pod的更新，如果发现有pod更新信息，则会自动在podWorker的同步周期中更新对应的pod



这正是Kubernetes实现中 "一切皆资源"的体现，即所有实体对象，消息都是作为etcd里保存起来的一种资源对待，其他所有组件间协作都通过基于APIServer的数据交换，组件间一种松耦合的状态。

#### 创建service

当客户端发起一个创建service的请求后，kubectl向APIServer的/service端点发送一个HTTP POST请求，请求的内容即客户端提供的service资源配置文件。



同样，APIServer收到该REST API请求后会进行一系列的验证操作。验证通过后，APIServer调用etcd的存储接口在后台数据库中创建一个service对象



kube-proxy会定期调用APIServer的API获取期望service对象列表，然后再遍历期望service对象列表。对每个service调用APIServer的API获取对应的pod集的信息，并从pod信息列表中提取pod IP和容器端口号封装成endpoint对象，然后调用APIServer的API在etcd中创建对象

在```userspace kube-proxy模式``下：

对于每个新建的service，kube-proxy会为其在本地随机分配一个随机端口号，并相应地创建一个ProxySocket，随后使用iptables工具在宿主机上建立一条从ServiceProxy到ProxySocket的了链路。同时，kube-proxy后台启动一个协程监听ProxySocket上的数据并根据endpoint实例的信息将来自客户端的请求转发给相应的service后端pod.



在```iptables kube-proxy模式```下：

对于每个新建的service，kube-proxy会为其创建对应的iptables。来自客户端的请求将由内核态iptables负责转发给service后端pod完成



最后，kube-proxy会定期调用APIService的API获取期望service和endpoint列表并与本地的service 和endpoint实例同步。

### Kubernetes 网络核心原理

#### 单pod单IP模型

Kubernetes为每一个pod分配一个私有网络地址段的IP地址，通过该IP地址，pod能够跨网络与其他物理机，虚拟机或容器进行通信，pod内的容器全部共享这个pod的容器配置，彼此之间使用localhost通信。

#### 单pod单IP实现原理

在每一个pod中有一个网络容器，该容器先于pod内所有用户容器被创建，并且拥有该pod的网络namespace，pod的其他用户容器使用Docker的--net=container:<id>选项加入该网络的namespace，这样就实现了pod内所有容器对于网络栈的共享

### Kubernetes 高级实践

应用健康检查:

* 进程级健康检查

* 业务级健康检查:

  活性探针：

    * HTTP Get: kubelet 将调用容器内Web应用的web hook 如果返回的状态为200 和 399 之间则成功
    * Container Exec: kubelet 将在用户容器内执行一次命令，返回码为0 则正常
    * TCP Socket : 尝试建立socker,但目前尚未支持

  如果readiness probe的健康检查结果是fail kubelet并不会杀死容器进程，而只是将该容器所属的pod 从endpoint列表删除

### Kubernetes 未来动向

坚持走更加开放的道路

汲取Borg与Omega的优秀设计思想

致力于树立行业标准