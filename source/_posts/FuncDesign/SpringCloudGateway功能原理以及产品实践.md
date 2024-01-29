---
title: SpringCloudGateway功能原理以及产品化实践
date: 2022-06-11 00:01:00
tags:
  - SpringCloudGateway
  - SCG
  - 云原生网关
categories: SpringCloudGateway
---

## 前言
近日，我司的项目总监让我们网关组的成员对企业级SpringCloudGateway网关相关内容进行整理，我正好借此机会沉淀一下这段时间以来对网关的开发实践以及思考。接下来我会按照以下内容进行介绍：
- 介绍企业级微服务开发面临的痛点、需求、场景和应用
- 介绍SpringCloud Gateway的核心原理
- 介绍SpringCloud Gateway在网关高可用、动态路由、可扩展插件、接口调试、限流熔断、性能优化等方面的实践与优化
- 介绍自己对企业级微服务网关的一些思考与未来的发展方向

## 微服务开发面临的问题

- 服务需要重复开发通用功能，随着微服务规模的不断扩大，这些校验冗余逻辑将越来越沉重，一旦校验规则有了变化，不得不去每个应用修改这些逻辑，增加了维护成本
- 随着时间的推移，可能需要改变系统目前的拆分方案，但如果客户端直接与微服务交互，强耦合，那么这种重构就很难实施
- 服务协议不统一，系统服务使用webService、gRPC以及其他RPC等非RESTFUL接口标准协议进行开发的应用，协议不统一，需要兼容
- 业务不断发展需求快速迭代，服务接口愈多，如何保护、监控、维护数以万计的接口

## 微服务网关的应用场景有哪些
- 微服务网关
- 业务系统集成
- 企业能力开放
- 接口生命周期管理
- 架构治理

## Spring Cloud Gateway的核心原理
Spring Cloud Gateway是Spring官方基于Spring5.0、SpringBoot2.0和Project Reactor等技术开发的网关旨在为微服务框架提供一种简单而有效的 统一的API路由管理方式，统一访问接口。Spring Cloud Gateway作为Spring Cloud生态体系中的网关，目标是替代Netflix的Zuul，其不仅提供统一的路由方式，并且
基于Filter链的方式提供了网关基本的功能，例如：安全、监控/埋点和限流等等。

Spring Cloud Gateway网关是一个内外衔接的数据交换组件，对内API接口的方式纳管所有要对外透出的微服务，作为出口端点，对外提供API接口给上游的Web应用、Mobile应用、外部微服务。网关核心在于将请求流量由上游发起经过网关到下游的微服务，在流量出入的过程中，网关在路由策略，协议转换、过滤、API组合等方面构建
网关的核心能力。

![scg核心原理.png](..%2F..%2Fimg%2Fscg%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%86.png)

### 关键术语

- 路由Route：即一套路由规则，是集URI、predicate、filter等属性的一个元数据类。
- 断言Predicate：Java8函数断言，这里可以看做是满足什么条件的时候，route规则进行生效。允许开发者去定义匹配来自于Http Request中的任何信息，如请求头和参数。
- 过滤器Filter：filter针对请求和响应进行增强、修改处理。filter可以认为是Spring Cloud Gateway最核心的模块，熔断、安全、逻辑执行、网络调用都是filter来完成的，其中又细分为gateway filter和global filter，区别在于是具体一个route规则生效还是所有route规则都生效。

![scg核心原理2.png](..%2F..%2Fimg%2Fscg%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%862.png)

## CSB2.0云原生网关如何保障服务高可用

项目术语：
- console: 网关配置控制台的云原生网关服务
- broker: 实际请求转发的云原生网关服务
- SLA:service-level agreement SLA的概念，对互联网公司来说就是服务可用性的一个保证。
- SLB:Server Load Balancing 指服务器负载均衡

网关承载着所有服务的入口流量,要求具备高可用的能力。而通常实现高可用的主要手段是数据的冗余备份和服务 的失效转移,而这两种手段在网关的具备体现为：
### 集群的部署结构：
CSB2.0云原生网关目前是基于Kubernetes容器编排平台进行部署，由云平台提供SLB作为流量到网关节点的负载均衡能力，网关根据服务器的性能进行集群的部署，这部分的SLA的能力由云平台负责保障。企业部署一个地方的网关节点集群，相当于单数据中心，也可根 据自己的需求部署多个网关节点集群，以达到多个数
据中心的容灾能力，这样部署就已经能保障网关的正常可用。如图所示：
![scg集群部署.png](..%2F..%2Fimg%2Fscg%E9%9B%86%E7%BE%A4%E9%83%A8%E7%BD%B2.png)

### 负载均衡的能力：
CSB2.0云原生网关实现了一套控制台即可对不同实例下的网关节点进行配置下发，并且同一个实例内的所有网关节点都是生效的。通常一个网关节点部署在一个服务器上，节点通过监听中间件(Redis、Nacos等)中的配置信息以此来获取、更新broker中的配置。此处的负载均衡是指broker节点对于下游服务的不同节点进行
请求转发，例如服务service1的请求会依次分发到服务下的节点node1、node2 和node3。如图所示：
![scg-lb.png](..%2F..%2Fimg%2Fscg-lb.png)

### 健康检查
请求到达服务节点前，尽管已经做了两层的负载，即请求到达云容器平台的负载均衡器以及broker对下游服务节点的负载。但是不管是网关broker节点，还是下游的某个节点发生了故障,请求还是有可能继续打到该节点上，这时候服务的最终结果仍是不可用。解决的方案是把所有有问题的的节点，包括broker节点、下游服务节点移出SLB的范围即可。因此需要解决的问题是如何知道broker节点以及下游服务的node节点是否正常？解决方案是对broker节点以及服务下游node节点进行健康检查：
- broker节点健康检查：CSB2.0云原生网关的部署依托于Kubernetes等容器编排平台，容器平台提供有多种对容器的健康检查方式，如容器探针、基于HTTP的探活检查
- 下游服务健康检查：可对下游服务设置正确的返回结果(请求状态码、超时期限等),定时轮训将请求打到下游服务，若发现返回异常且满足从负载节点移除的条件，则将该节点移除。当下游服务节点恢复再加回负载列表。

### 节点恢复
CSB2.0云原生网关依托于Kubernetes云容器编排平台进行部署，节点的自恢复能力由云平台保证。

### 熔断与降级
下游服务可能会出现一些超出预期的错误，这种错误有可能影响到系统的正常运行，比如请求引发的阻塞，这种请求阻塞有可能占用系统宝贵的资源，如：内存、线程、数据库等，消耗的资源有可能会拖垮整个系统，因此网关
需要判断服务不可用就切断对服务的访问,CSB2.0云原生网关底层采用阿里开源的sentinel框架作为流量控制、熔断降级、系统负载保护等多个纬度保障服务的稳定性。
当网关请求在一段时间内失败次数达到一定条件，就会触发熔断。目前CSB2.0支持的熔断条件有：
1、后端响应时间
2、后端错误码
3、触发熔断的请求阈值
降级策略目前支持Mock的形式返回响应参数，包括响应码、响应头、响应体。
当发生熔断后，判断请求是否恢复正常的条件，若连续请求成功次数达标，则恢复转发，服务自动转入监控期；否则，继续进入熔断期。如此反复。如下图：
![熔断.png](..%2F..%2Fimg%2F%E7%86%94%E6%96%AD.png)

### 接口重试
虽然有很多机制保障接口的可访问，但是一个请求报错的原因有很多，偶然一次报错不一定是服务不可用，最简单的，第一次不行，应该再访问一次或几次，以确定结果。 请求重试可以说是网关对接口转发的基本要求，每个接口都应该可以设置重试次数。当请求失败后，网关应立即再次请求，直到拿到正常返回，或是达到重试阈值，再将结果返回给客户端。

## CSB2.0云原生网关如何实现动态路由
为了应对网关路由各种复杂业务场景，要求网关在不重启服务的情况下，实现对API路由规则的动态配置，实时生效。SpringCloud提供有两种原生动态路由的方式：
- Spring Cloud DiscoveryClient原生支持：
  Spring Cloud原生支持服务自动发现并且注册到路由之中，通过在application.properties中设置spring.cloud.gateway.discovery.locator.enabled=true, 同时确保DiscoveryClient的实体(Nacos，Netflix Eureka, Consul, 或 Zookeeper) 已经生效，即可完成服务的自动发现及注册。
- 一种是基于Actuator API
  SpringCloud Gateway提供有OpenAPI来建立路由信息，请求内容为JSON请求体，请求方法为POST 如路径：/gateway/routes/{id_route_to_create}

以上路由扩展的自由度有限，第一种方式的服务都要依托与SpringCloud家族体系下，第二种无法满足高度定制化的需求。CSB2.0的做法是对SpringCloud Gateway做了底层修改，扩展了Spring Cloud Gateway底层路由加载机制，将Spring Cloud Gateway运行态时保存的路由关系，通过实现、继承加载自定义类的方式，对其进行动态路由修改，每当路由有变化时，再触发一次动态的修改。

因此这种实现需要两种保障：
1、监听机制
2、实现自定义路由的核心类
Spring Cloud Gateway 核心加载机制如图所示：

![scg配置监听.png](..%2F..%2Fimg%2Fscg%E9%85%8D%E7%BD%AE%E7%9B%91%E5%90%AC.png)

## CSB2.0云原生网关如何实现插件热插拔

CSB2.0插件热插拔还在排期开发，但可参考 https://juejin.cn/post/6963453967497953311

## CSB2.0云原生网关如何实现接口调试
CSB2.0需要Console对Broker进行接口调试，但在专有云网络下存在【管控区】和【用户区】的区分，两者之间存在网络通信限制，具体表现为
- 用户区访问管控区，可以通过 vip 打通
- 管控区不允许访问用户区

这里的“访问”在 TCP 层面可以理解为不允许主动建立连接，而 API 调试的需求从功能层面来看，的确是【管控区】访问【用户区】，在无法主动建立连接的情况下，需要设计网络反向访问方案。

本方案提出一个设计，由【用户区】的 broker 主动建立 TCP 长连接到【管控区】的 console，借助于 TCP 双工通信的特性，console 可以通过持续维护 broker 的连接，从而完成与 broker 的通信。
整体设计如图所示：
![接口调试整体方案.png](..%2F..%2Fimg%2F%E6%8E%A5%E5%8F%A3%E8%B0%83%E8%AF%95%E6%95%B4%E4%BD%93%E6%96%B9%E6%A1%88.png)

### VIP连接方案
长连接实现方案使用 WebSocket实现 console 与 broker 的交互。但是在实际部署中，【用户区】的 broker 经过了一个 VIP 连接到【管控区】的 console，由于 VIP 随机负载均衡的特性的，可能会出现部分 console 没有持有长连接的问题，当这些 console 节点接受请求之后，将会无法处理。如图所示：

![scg-vip问题.png](..%2F..%2Fimg%2Fscg-vip%E9%97%AE%E9%A2%98.png)

为了解决上述问题，如下图引入心跳机制。broker 通过定时任务向 VIP 发送建连请求，console 在接受到请求之后，需要返回 console ip，broker 在接收到 console ip 的响应之后，
需要比对 console ip 与本地的 console ip 缓存池，从而判断是否是一条新的连接
- 从连接层面来看，broker 与 vip 建立了重复的连接，但实际上连接到了不同的 console 实例
- 轮询机制保障了最终能够与所有 console 实例建立连接
- 轮询机制兼容了 console 动态扩缩容的场景
- 轮询机制兼容了连接断开的场景

![接口调试最终设计.png](..%2F..%2Fimg%2F%E6%8E%A5%E5%8F%A3%E8%B0%83%E8%AF%95%E6%9C%80%E7%BB%88%E8%AE%BE%E8%AE%A1.png)

## CSB2.0云原生网关如何实现多协议转换
在SpringCloud-Gateway网关中扩展多种协议如：Grpc、WebService、Dubbo、HSF等协议，在此之前我们需要分析一下SpringCloud-Gateway filter机制。springcloud-gateway基于过滤器实现， 分为pre和post两种类型的过滤器，分别处理前置逻辑和后置逻辑。客户端将http请求到gateway，请求经过前置过滤器处理后转发到具体到业务服务中，收到业务服务的相应
后，响应将通过后置过滤器处理后返回客户端，其中过滤器的处理顺序按照order排序（后置处理器倒序排序）;
![scg-filter流程.png](..%2F..%2Fimg%2Fscg-filter%E6%B5%81%E7%A8%8B.png)

对于http-http的请求代理来说，NettyRoutingFilter是作为filter chain的最后一个pre filter，它负责将请求转发到具体的业务服务中
```java
public class NettyRoutingFilter implements GlobalFilter, Ordered {
  
    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```
NettyWriteResponseFilter作为post filter chain的第一个filter，它负责将业务服务的响应返回给客户端
```java
public class NettyWriteResponseFilter implements GlobalFilter, Ordered {

  public static final int WRITE_RESPONSE_FILTER_ORDER = -1;

  @Override
  public int getOrder() {
    return WRITE_RESPONSE_FILTER_ORDER;
  }
}
```
因此要实现http作为入口协议的多协议的泛化调用，我们需要在pre filter chain中添加一个filter代替NettyRoutingFilter，负责将http编码为其他协议的请求，同时在post filter chain中添加一个filter代替NettyWriteResponseFilter，负责将其他协议的响应转换为http响应。
以dubbo的泛化调用为例，实现如下图所示：
![scg泛化调用.png](..%2F..%2Fimg%2Fscg%E6%B3%9B%E5%8C%96%E8%B0%83%E7%94%A8.png)

## 对企业级微服务网关的一些思考与未来的发展方向
我们可以看到Spring Cloud Gateway可以很好地与其背后的Spring 社区和 SpringCloud 微服务体系有着很好的适配和集成，这与 Java 语言流行的原因如出一辙。如果一个企业主打的技术栈是Java 体系，那么基于SpringBoot/ SpringCloud 开发微服务，选型 SpringCloud Gateway 作为微服务网关，会有着得天独厚的优势。
而在企业级微服务网关的探索上，我认为可以基于一下几点进行后续的演进：
- 多协议入口网关建设：基于scg作为网关引擎更多使用场景为南北向的流量转发，但是在企业级网关建设中，往往需要支持多种协议的入口，如：dubbo、grpc以及私有协议等东西向流量的转发，因此需要在scg的基础上进行扩展，支持多种协议的入口。
- 性能与稳定性建设: 优化Spring Cloud Gateway的性能，确保在高并发情况下的稳定性和低延迟。
- 云原生支持: 加强与Kubernetes的集成，支持容器化和微服务自动化部署。
- 云服务厂商整合: 提供对主要云服务厂商的无缝整合，如AWS, Azure, GCP等。