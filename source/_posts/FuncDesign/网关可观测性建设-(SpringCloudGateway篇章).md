---
title: 网关可观测性建设-(SpringCloudGateway篇)
date: 2022-05-14 00:01:00
tags:
  - SpringCloudGateway
  - 可观测性
  - 云原生网关
categories: 云原生网关
---

## 前言
可观测性体系的建设，可以说是很多开源产品距离企业级使用的距离，SpringCloud Gateway 亦是如此。本文将从可观测性的角度，对SpringCloud Gateway进行一些改造，以期达到企业级的可观测性要求。

## 可观测性
网关通常会需要记录三类可观测性指标

- Metrics：如上图所示，记录请求数、QPS、响应码、P99、P999 等指标

- Trace：网关链路能够串联后续微服务体系链路，实现全链路监控

- Logging：按类别打印网关日志，常见的日志分类如 accessLog、requestLog、remotingLog 等
### Metrics
开源 SpringCloud Gateway 集成了 micrometer-registry-prometheus，提供了一个开箱即用的大盘：https://docs.spring.io/spring-cloud-gateway/docs/3.1.8/reference/html/gateway-grafana-dashboard.json
需要更加丰富维度的指标则需要自行埋点。

而在展示上则是通过前端内嵌 Grafana 来实现，具体的功能架构如下图


### Trace
Trace 方案推荐对接 opentelemetry。

### Logging
Logging 方案则是 SpringCloud Gateway 开源欠缺的，在实际生产中至少应该打印 accessLog 记录请求信息，按需开启 requestLog 记录请求的 payload 信息和响应体信息，以及与后端服务连接的日志，用于排查一些连接问题。日志采集方案我们的实践是将 accessLog 输出到标准输出中，方便在 K8s 架构下配置采集，或者采用日志 agent 的方案进行文件采集。


