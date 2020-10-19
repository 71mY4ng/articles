---
title: SOA 和 微服务
tags: SOA, 微服务, Microservice
notebook: IT剪藏
---

## SOA

SOA (Service-oriented-architecture) 是一种架构设计模式

按业务拆分系统，模块相互独立

模块提供了API，即服务，模块间互相通信，完成业务

服务治理，服务的规模达到一定程度后，服务和调用方间的关系就需要管理。dubbo + zookeeper, SpringCloud 都提供了服务治理的功能。

在此基础上衍生了许多规范：

* SOAP: http+xml
* REST: http+json
* RPC: sockect

## Microservice

微服务与 SOA 相似，都是以松耦合，可复用，单一职责模块等特性组成纲领的。

但微服务与 SOA 之间的区别是其作用的域，SOA 是用于企业应用之间的，而微服务是用于一个应用的。

也就是说，对于一个应用进行模块拆分，使得其成为松耦合，模块间独立并可复用，弹性且可扩展的敏捷分布式架构，就是对单体应用进行微服务化的过程。

## 细节对比

### 1. 同步调用

SOA 主要使用的是同步调用的协议，例如 RESTful API

微服务间的通信更倾向于异步通信，例如事件驱动，这方面 pub/sub 应用得较多。异步通信相比同步通信来说更具有弹性，低延迟的特性。



__参考：__

---
* [IBM: SOA vs Microservices](https://www.ibm.com/cloud/blog/soa-vs-microservices)