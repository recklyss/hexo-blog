---
title: Scaffold-Cloud —— SpringCloud微服务架构实践
date: 2019-11-12 13:42:24
categories: [SpringCloud]
tags: [SpringCloud, 微服务, 分布式, SpringBoot]
top: true 
---

<center><u style="font-size=16px;"><a href="/">Scaffold-Cloud: SpringCloud微服务脚手架</a></u></center> [幕布](https://mubu.com/doc/6NZlNw3DIw)

Scaffold-Cloud 是一个适用于开发者学习的 Spring-Cloud 微服务项目脚手架。项目期望集成大部分目前互联网公司使用的主流的Spring-Cloud微服务相关工具和服务。并结合一些实际的业务增加一些其他功能，如：分布式事务、定时任务、消息队列、日志分析等等，然后加入 CI/CD 并引入 docker 部署。

Scaffold-Cloud 基于 SpringCloud Netflix 全家桶进行微服务项目的构建，所以在这之前，使用 Scaffold-Cloud 需要先了解下 SpringCloud 以及 Netflix 工具全家桶。

![](netflix.png)

<center><u style="font-size=6px; color:gray">Sorry, NetFlix is not available in your country yet.</u></center>

<!--more-->

## Spring Cloud 介绍 

#### 从单体应用到微服务

在早期的企业中，项目基本上都是单体应用，将一个网站部署在一台单独的服务器上对用户提供服务。但是这样的服务最大的缺点就是，当服务器断电或者服务进程挂掉，用户直接就无法访问。后来演进出集群服务，将同样的服务在多台服务器分别部署，使用负载均衡等手段让服务对于用户来说看到的都是同一个，这样当一台服务器夯机至少还有其他的服务器提供相同的应用。

但是，当企业级服务变得越来越复杂的时候，项目变得越来越庞大，有时甚至影响到了服务的部署，这个时候就应该将庞大的服务拆分成多个子系统，部署在不同的服务器上，这样的服务当遭遇高并发访问的时候也能够将请求压力分摊到多个服务器上，这就是分布式。

而当企业需要开启一个新的项目时，为了避免重复造轮子，往往一些可复用的模块会被拆分出来作为一个微小的系统，它可以独立的开发、设计、运行和运维，只需要使用 ESB企业服务总线将所有服务整合并提供统一的访问入口，就能够复用，这就是 SOA。

而微服务是对以上服务架构的最终演进结构：将服务切分成多个微小的应用，提供统一的服务访问协议HTTP(SpringCloud)/TCP(Dubbo)，强调运行时的分散解耦，在业务上也有着高度的抽离。

> `微服务架构风格`**是一种将`一个单一应用程序`开发为`一组小型服务`的方法，每个服务运行在自己的进程中，服务间通信采用轻量级通信机制**(通常使用HTTP资源API)。这些**服务围绕`业务能力`构建**，并且可通过**全自动部署机制独立部署**。这些服务共用一个**最小型的集中式的管理**，服务可用不同的语言开发，使用不同的**数据存储技术**。
> ![](microservice.webp)
> —— Martin Fowler

**但是拆分成这么微小的微服务一定会碰到各种各样的问题——如何拆分？服务之间如何发现？如何授权？如何做负载均衡？如何管理多服务配置？如何跟踪调用链路？如何实时查看服务状态等等... ... SpringCloud 就是以上一系列问题的 Solver。它提供了一整套的解决方案！**

#### SpringCloud

首先，SpringCloud 并不是一个框架，而是一个微服务的规范，或者说是一个微服务工具集合。

SpringCloud特点：

- 约定大于配置，基于 SpringBoot
- 开发部署于各种环境，AWS，阿里云，PC 等
- 隐藏组件复杂性，声明式配置，无 xml
- 开箱即用，快速启动
- 丰富的轻量级组件
- 灵活选型，如注册发现可用 eureka，zookeeper 或者 Redis

### SpringCloudNetFlix 全家桶





## 项目创建的目的？

第一个目的是为了本人能够熟悉和学习 Spring-Cloud 相关知识，不过在做了一段时间之后还是希望能够分享出来让更多的同学一起学习和讨论。



## 项目结构如何？





## 项目已经集成了哪些功能？

- [RocketMQ]([http://zhangjiaheng.cn/blog/20190819/%E5%A6%82%E4%BD%95%E5%9C%A8Spring%20Cloud%E4%B8%AD%E4%BC%98%E9%9B%85%E7%9A%84%E4%BD%BF%E7%94%A8Rocket%20MQ/](http://zhangjiaheng.cn/blog/20190819/如何在Spring Cloud中优雅的使用Rocket MQ/))
- [分布式事务]([http://zhangjiaheng.cn/blog/20190806/%E6%88%91%E9%A1%B9%E7%9B%AE%E4%B8%AD%E4%BD%BF%E7%94%A8%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1%EF%BC%9ALCN/](http://zhangjiaheng.cn/blog/20190806/我项目中使用的分布式事务：LCN/))
- 可视化JOB





## 如何快速开始？

### 1. 本地直接启动

####	1.1 所需环境

#### 1.2 启动流程/顺序



### 2. 使用 docker 部署



## 项目未来还需要做什么？







---

#### 参考：

> https://juejin.im/post/5c28f2fe51882565a15776fb

