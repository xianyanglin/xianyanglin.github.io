---
title: 关于Hango网关中gportal和api-plane组件合并实施方案
date: 2023-03-12 00:01:00
tags:
  - Hango
  - SCG
  - 云原生网关
categories: Hango
---
# 组件合并实施方案
## 背景
网关为了降低部署成本，减少运维代价,现需要将gportal和api-plane的组件在部署的时候进行合并，精简部署的组件。以下将围绕实施方案以及需要考虑的相关问题进行展开。
方案 不涉及到代码模块的合并，在K8s基座内部的服务模块合并方案不外乎两种，一是将portal和api-plane部署在pod的同一容器进程内，二是将portal和api-plane放在同一pod的不同容器中。
但是第一种方案将两个服务放在同一个容器中既不符合Kubernetes的最佳实践，同时也存在诸多问题，诸如：
* 它们的部署和管理变得更加复杂，对于不同的服务可能需要不同的配置和更新策略，这会增加容器的复杂度和维护成本。
* 共享同一个资源池，这可能会导致它们之间的资源争用或者不均衡，难以对它们进行单独的扩展和调整。
* 共享同一个进程空间和文件系统，缺乏隔离性，这可能会导致它们之间的冲突或干扰等等。
  而将gportal和api-plane合并至同一pod的不同容器中可以最大复用原先各自服务的deploy里的配置内容，且不需要更改镜像的配置信息，可以较小工作量完成组件的聚合。
  基于以上诸多原因我们采用第二种方案将gportal和api-plane合并至同一pod的不同容器中。
## 相关问题
gportal和api-plane合并至同一pod的不同容器中同时也需要考虑以下问题：
* 资源占用增加
  原先将其分开部署在不同的deploy中，我们可以按需控制gportal和api-plane各自的副本数，但现在两个服务的容器数量比为```1:1```，资源开销将有增加。
* gportal和api-plane之间的通信问题：

  1、api-plane的服务地址是存储在DB的hango_gateway表中，我们可以通过更改表数据来指定api-plane的服务地址。

  2、同一pod的不同容器共享同一个进程空间和网络命名空间，这意味着它们将共享相同的主机名、IP地址和端口号。我们可以指定api-plane的服务地址为localhost进行调用。

* 健康检查
  我们在做组件合并的过程中会为各个容器服务配置健康检查， 当pod内所有容器的健康检查都成功时，才会将流量路由到Pod，这就要求gportal和api-plane服务必须同时可用。
## 如何兼容多集群纳管
现阶段网关是依靠于gportal调用不同环境乃至不同集群的api-plane地址实现多网关的纳管，每个api-plane的地址所指向的上游集群都享有同一份网关资源。
![现阶段多网关纳管.png](..%2F..%2Fimg%2FfuncDesign%2F%E7%8E%B0%E9%98%B6%E6%AE%B5%E5%A4%9A%E7%BD%91%E5%85%B3%E7%BA%B3%E7%AE%A1.png)
如将部署架构下api-plane和gportal合为同一个pod中，两者同处于管控集群，意味着原先多网关的纳管能力需要从gportal下沉到 api-plane服务中，由api-plane为共享同一份资源的计算集群进行分组。下面我们会以较多篇幅讨论多集群纳管的方案

### 多集群纳管方案一：

由前文我们了解到在新的部署架构中，gportal和api-plane同处于管控集群，我们需要将原先DB存储api-plane服务的地址指向改为 localhost，并且表并且gportal所有对于api-plane的调用都需要传递共享该资源的集群分组标识，由api-plane根据根据分组标识筛选出对应计算集群的信息对其资源进行操作。

![多集群纳管方案一.png](..%2F..%2Fimg%2FfuncDesign%2F%E5%A4%9A%E9%9B%86%E7%BE%A4%E7%BA%B3%E7%AE%A1%E6%96%B9%E6%A1%88%E4%B8%80.png)
api-plane配置文件新增了对相同副本的集群进行了分组
```yaml
k8s:
  groups:
    - group : gateway1
      clusters:
        master:
          k8s-api-server: ""
          cert-data: ""
          key-data: ""
        slave1:
          k8s-api-server: ""
          cert-data: ""
          key-data: ""
        slave2:
          k8s-api-server: ""
          cert-data: ""
          key-data: ""
    - group : gateway2
      clusters:
        master:
          k8s-api-server: ""
          cert-data: ""
          key-data: ""

```


需要改造内容

1、api-plane服务deploy、ConfigMap 的yaml文件内容迁移至gportal的deploy containers节点下

2、DB的```hango_gateway```表中，我们更改表数据来指定api-plane的服务地址为localhost。

3、chart场景化抽象

4、gportal对api-plane发送的请求都需要携带group信息

5、api-plane的ConfigMap中配置多集群的api-server地址，服务读取ConfigMap初始化集群客户端，并根据group进行客户端缓存

6、需要操作api-server时，从请求中获取group信息，再从缓存中根据group信息获取Kubernetes 客户端

### 多集群纳管方案二：
根据前文内容，若gportal和api-plane位于同一管控集群中，那么api-plane就需要承担管理多集群配置下发的责任，这将使得API-plane相关配置下发的实现变得更为复杂。但是，我们可以稍微思考一下，如果将gportal和api-plane都部署在计算集群中，也可以在保持现有部署架构的情况下实现多集群纳管。

![多集群纳管方案二.png](..%2F..%2Fimg%2FfuncDesign%2F%E5%A4%9A%E9%9B%86%E7%BE%A4%E7%BA%B3%E7%AE%A1%E6%96%B9%E6%A1%88%E4%BA%8C.png)
### 多集群纳管方案三：
以上都是改动比较大的改造，我们也可以不用走一刀切的策略。我们可以对不同场景的chart进行抽象。
如果数据面和管控面都处于同一个集群下，那么我们可以考虑portal和api-plane组件在deploy中进行合并。

![多集群纳管方案三-1.png](..%2F..%2Fimg%2FfuncDesign%2F%E5%A4%9A%E9%9B%86%E7%BE%A4%E7%BA%B3%E7%AE%A1%E6%96%B9%E6%A1%88%E4%B8%89-1.png)
如果数据面和管控面不在同一个集群下，多集群还是按照现在portal和api-plane组件分离的方式：

![多集群纳管方案三-2.png](..%2F..%2Fimg%2FfuncDesign%2F%E5%A4%9A%E9%9B%86%E7%BE%A4%E7%BA%B3%E7%AE%A1%E6%96%B9%E6%A1%88%E4%B8%89-2.png)
## 后续长期规划
尽管gportal和api-plane的服务职责不同，但它们都属于Java服务的体系都由网关团队进行开发，且api-plane仅被gportal服务所调用，现无论它们在同一pod还是分别拆分到计算管控集群，只要它们作为两个服务存在，就会造成资源损耗。因此，我们可以考虑将它们合并为同一个工程的不同模块，按职责划分模块间关系。下图为portal现有的功能模块，未来，可以将API-plane工程作为API-Server的职责模块迁移至portal工程中，这也是合理的。
![组件代码合并.png](..%2F..%2Fimg%2FfuncDesign%2F%E7%BB%84%E4%BB%B6%E4%BB%A3%E7%A0%81%E5%90%88%E5%B9%B6.png)

这样将两个相互调用的微服务合并为同一个工程的两个 model，会带来以下好处：
* 减少通信成本：将原本需要通过网络通信的两个微服务合并到同一个工程中，可以减少网络通信的成本和延迟。
* 简化部署操作：将两个微服务合并为同一个工程，可以简化部署操作，减少部署时间和错误。
* 提高程序的可维护性：将两个微服务合并为同一个工程，可以减少代码的重复，降低维护成本。
* api-server与DB之间可以通过事务保证资源数据的一致性

在考虑将两个微服务合并为同一个工程时，api-plane所有的controller都将变成普通方法由portal进行调用，存在相当大的改造以及测试成本。
![前端管控多集群.png](..%2F..%2Fimg%2FfuncDesign%2F%E5%89%8D%E7%AB%AF%E7%AE%A1%E6%8E%A7%E5%A4%9A%E9%9B%86%E7%BE%A4.png)

## 结论
综上为未来网关组件合并的探索，但现阶段标品如果按照以上的内容进行部署配置上的更改会给SRE带来很大的部署成本且原先的部署helm脚本会变得更复杂，因此我们前期先在开源版本进行组件的合并，后续在完成包括服务模块合并到portal服务中之后，我们再将helm脚本迁移到标品的版本。