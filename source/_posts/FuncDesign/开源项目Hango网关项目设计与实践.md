---
title: 开源项目Hango网关项目设计与实践
date: 2023-07-14 00:01:00
tags:
  - Envoy
  - 网易
  - 云原生网关
categories: 云原生网关
---

## 前言
Hango 是由网易公司开源的一个基于Envoy构建的高性能、可扩展功能丰富的云原生API网关，本文将从Hango的诞生背景、架构设计、功能特性、大规模落地等方面进行介绍。

## 诞生背景
Envoy起初在网易轻舟是作为服务网格Istio的数据面使用，承担了东西向、南北向全部数据流量的代理、治理与观测职责。随着服务网格在网易内部大规模落地，我们对 Envoy 的功能、性能、扩展性、可观测性等多方面有了全面的研究与实践，也深刻感受到 Envoy 优质的内在品质，及其在云原生时代巨大的发展潜力。
于是我们开始尝试基于 Envoy 建设网易的新一代 API 网关，目标是替换网易内部较多业务采用 Java 异步化网关、 Kong 网关，并能够满足业务逐步进入云原生时代的南北向流量治理需求。从结果上看，选型 Envoy 不仅让我们顺利实现了网易 API 网关全面升级，还推动了网易云原生、微服务技术栈整体的统一与向前发展。

## 架构设计
Hango 基于云原生理念构建，数据面基于Envoy进行扩展，增强插件链，提供 Rider 模块用于自定义插件扩展；控制面组件包括 Slime，Istio，API Plane以及Portal模块。其架构图如下所示：
![hango设计架构.png](..%2Fimg%2Fhango%E8%AE%BE%E8%AE%A1%E6%9E%B6%E6%9E%84.png)
组件说明：
- Envoy: Hango网关的数据面流量入口，通过XDS模式获取Istiod配置
- Istiod: Hango网关对接Envoy的适配层，监听指定CRD并通过XDS向Envoy下发配置
- Slime: 提供指定Slime CRD，以支持Hango支持插件、限流、服务发现等特性
- Hango Portal: Hango对接前端界面的Web层模块，存储Hango业务数据
- Hango Api-plane: 对接Hango Portal，管理Hango CR声明周期
- Hango UI: Hango前端界面，提供Hango配置、观测、调试等功能

Hango UI将Envoy的功能进行了可视化的配置，与Hango Portal进行交互，Hango Portal将业务信息存于DB，并调用Hango Api-plane生成服务路由相应的CR，Istiod在Watch到API-plane生成的CR会通过XDS协议下发给数据面Envoy，Envoy将配置信息应用到数据面。
对于插件的能力的扩展，我们使用自定义的CRD(EnvoyPlugin)来承载插件信息，Slime模块监听Slime CRD，将Slime CRD转换为EnvoyFilter的插件配置对数据面envoy进行扩展。

下面是Hango的数据流向：


## 功能特性

### 虚拟网关

#### 虚拟网关
在L7层通用网关中，一组Envoy网关集群可以表现出不同的形态，如API网关、L7负载均衡、ingress。网关集群在部署时已确定的一组网关实例，称之为网关，不同表现形态称之为虚拟网关。 依托于网关，可通过不同场景进行配置。网关数据层采用Envoy作为底层代理，通过不同的Listener配置开放Envoy不同的监听端口，对应不同形态的虚拟网关。在L7层通用网关中，一组Envoy网关集群可以表现出不同的形态，如API网关、L7负载均衡、ingress。网关集群在部署时已确定的一组网关实例，称之为网关，不同表现形态称之为虚拟网关。 依托于网关，可通过不同场景进行配置。网关数据层采用Envoy作为底层代理，通过不同的Listener配置开放Envoy不同的监听端口，对应不同形态的虚拟网关。

以下为虚拟网关的功能架构图：
![虚拟网关功能架构图.png](..%2Fimg%2F%E8%99%9A%E6%8B%9F%E7%BD%91%E5%85%B3%E5%8A%9F%E8%83%BD%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

一组Envoy网关实例部署在Kubernetes集群中，用户通过配置服务在网关生成Listener(虚拟网关)，Listener(虚拟网关)监听对应的服务，当有请求访问该监听端口时，进行流量转发。

#### Kubernetes Gateway
Gateway API作为Kubernetes入口网关的最新成果，得到行业的广泛支持。它代表了Ingress功能的一个父集，定义了一系列以Gateway资源为中心的资源集合。与Ingress类似，Kubernetes 中没有内置Gateway API默认实现，需要依赖基础设施商提供Gateway Class。 

官方文档见：https://gateway-api.sigs.k8s.io

作为Ingress资源的升级，Gateway API提供了一系列治理能力更强、表达性更优、可扩展性更高的资源集合，其中GatewayClass、Gateway和HTTPRoute已经进入Beta阶段，其他CRD还处于实验阶段。 此处只需要关注Gateway和xRoute资源，详细的API定义可参考Gateway API

以下图片为Gateway API的相关组件：
![Gateway API相关组件.png](..%2Fimg%2FGateway%20API%E7%9B%B8%E5%85%B3%E7%BB%84%E4%BB%B6.png)

- GatewayClass：GatewayClass是由基础架构提供商定义的集群范围的资源，该资源用于指定对应的Gateway Controller。目前已实现的Gateway Controller的产品包括Envoy Gateway（beta）、Istio（beta）、Kong（beta）等，详情可参考Gateway Controller

- Gateway： 核心网关资源，主要规范了以下三部分内容：

    - Listeners：网关监听器列表，每个监听器都代表了一组主机名、端口、协议配置。

    - GatewayClassName：用于指定生效的GatewayClass。

    - Address：定义网关代理的请求地址。

- xRoute： 代表需要不同特性协议的路由资源，每种协议路由都有特定的语义，这种模式具有较好的扩展性，例如可以定义DubboRoute、gRPCRoute等。每种资源都定义了基本的匹配、过滤和路由规则，这些规则只有被绑定到相应的Gateway资源上才可以生效。目前只有HTTPRoute进入Beta阶段。

以下为Kubernetes Gateway在hango的功能实现架构图，Gateway API在控制台被创建之后会被开源的Istio控制面将Gateway API对象转换为Istio API对象，最终下发至Envoy数据面。 hango在此基础之上对HttpRoute做了插件上的增强，提供了更多丰富的插件能力。

![Kubernetes Gateway技术架构图.png](..%2Fimg%2FKubernetes%20Gateway%E6%8A%80%E6%9C%AF%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

#### Kubernetes Ingress
Ingress是K8s生态中定义流量入口的一种资源，但其只是个规则，仅创建Ingress资源本身是没有任何效果的，想让其生效，就需要有一个Ingress Controller去监听K8s中的ingress资源， 并对这些资源进行规则解析，转换为其数据面中的代理规则，并由数据面来进行流量转发。当前K8s默认的Ingress Controller实现是Nginx，本次方案描述如何将通用网关纳管Ingress的流量。

Ingress资源是对集群中服务的外部访问进行管理的 API 对象，可以将集群内部服务通过HTTP或HTTPS暴露出去，流量路由规则由Ingress资源定义。
![Ingress.png](..%2Fimg%2FIngress.png)

目前只支持Ingress v1版本流量纳管，v1版本的Ingress资源定义如下：
```yaml
##ingress v1
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
   annotations:
      kubectl.kubernetes.io/last-applied-configuration: |
         {"apiVersion":"networking.k8s.io/v1","kind":"Ingress","metadata":{"annotations":{"kubernetes.io/ingress.class":"istio"},"name":"test","namespace":"hango-system"},"spec":{"ingressClassName":"istio","rules":[{"http":{"paths":[{"backend":{"service":{"name":"istio-e2e-app","port":{"number":80}}},"path":"/get","pathType":"Prefix"}]}}]}}
      kubernetes.io/ingress.class: hango
      skiff.netease.com/project: hango
   creationTimestamp: "2023-07-14T07:33:17Z"
   generation: 7
   name: test
   namespace: hango-system
   resourceVersion: "9320368"
   labels:
     istio.io/rev: gw-1.12
   uid: 1ba7e839-da43-4c00-afeb-85d173911003
spec:
   rules:
     - http:
        paths:
          - backend:
              service:
              name: istio-e2e-app
              port:
              number: 80
            path: /get
            pathType: Prefix
status:
  loadBalancer:
    ingress:
      - ip: xxx.xxx.xxx.xxx
```
Ingress v1版本的资源定义中，主要包含以下几个重要注解：
- istio.io/rev：指定网关版本，目前只支持gw-1.12
- kubernetes.io/ingress.class：指定ingress的class，目前固定为hango
- skiff.netease.com/project：指定项目ID，目前固定为hango
- spec.rules：指定ingress的路由规则，目前只支持http协议
- status.loadBalancer.ingress：指定ingress的访问地址
### 服务发现
目前Istio1.8之后只保留了对接Kubernetes注册中心的逻辑，但是在实际生产实践中，我们发现大量用户只是将Kubernetes作为部署和管理的平台，服务信息依旧注册在第三方服务注册中心，如Nacos和Zookeeper，因此我们必须解决对接第三方注册中心的问题，以满足用户的需求。
我们采用的方案是网易开源的slime组件作为服务发现的适配层，slime组件支持对接多种第三方注册中心，如Nacos、Zookeeper、Eureka、Consul等，同时支持对接Kubernetes注册中心，将服务发现的逻辑统一抽象为slime CRD，通过slime CRD将服务发现的逻辑下发至Envoy数据面。

slime组件的相关介绍见：[Slime化解服务网格多注册中心兼容之痛](https://slime-io.github.io/blog/Slime%E5%8C%96%E8%A7%A3%E6%9C%8D%E5%8A%A1%E7%BD%91%E6%A0%BC%E5%A4%9A%E6%B3%A8%E5%86%8C%E4%B8%AD%E5%BF%83%E5%85%BC%E5%AE%B9%E4%B9%8B%E7%97%9B/)

## 大规模落地
本文结合网易内部的实践经验，将Hango的大规模落地分为三个角度介绍Hango网关在网易集团内部的落地实践。

### 平滑纳管
基于Hango网关规模落地中，我们对容器服务，裸金属服务等实现了平滑纳管以及平滑迁移，为业务上云提供了便利接入，打消了业务上云无法平滑的顾虑。

在集团内部，针对不同的业务划分不同的网关集群，从业务上进行配置隔离，提升网关可靠性。
![平滑纳管-集群隔离.png](..%2Fimg%2F%E5%B9%B3%E6%BB%91%E7%BA%B3%E7%AE%A1-%E9%9B%86%E7%BE%A4%E9%9A%94%E7%A6%BB.png)
### 灰度发布

### 可观测

## 写在最后
