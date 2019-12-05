---
title: Scaffold-Cloud —— SpringCloud微服务架构实践
date: 2019-10-25 13:42:24
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

##### SpringCloud特点：

- 约定大于配置，基于 SpringBoot
- 开发部署于各种环境，AWS，阿里云，PC 等
- 隐藏组件复杂性，声明式配置，无 xml
- 开箱即用，快速启动
- 丰富的轻量级组件
- 灵活选型，如注册发现可用 eureka，zookeeper 或者 Redis

##### SpringCloud 各版本组件及[版本兼容性](https://spring.io/projects/spring-cloud)

| Component                 | Edgware.SR6    | Greenwich.SR2 | Greenwich.BUILD-SNAPSHOT |
| ------------------------- | -------------- | ------------- | ------------------------ |
| spring-cloud-aws          | 1.2.4.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-bus          | 1.3.4.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-cli          | 1.4.1.RELEASE  | 2.0.0.RELEASE | 2.0.1.BUILD-SNAPSHOT     |
| spring-cloud-commons      | 1.3.6.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-contract     | 1.2.7.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-config       | 1.4.7.RELEASE  | 2.1.3.RELEASE | 2.1.4.BUILD-SNAPSHOT     |
| spring-cloud-netflix      | 1.4.7.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-security     | 1.2.4.RELEASE  | 2.1.3.RELEASE | 2.1.4.BUILD-SNAPSHOT     |
| spring-cloud-cloudfoundry | 1.1.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-consul       | 1.3.6.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-sleuth       | 1.3.6.RELEASE  | 2.1.1.RELEASE | 2.1.2.BUILD-SNAPSHOT     |
| spring-cloud-stream       | Ditmars.SR5    | Fishtown.SR3  | Fishtown.BUILD-SNAPSHOT  |
| spring-cloud-zookeeper    | 1.2.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-boot               | 1.5.21.RELEASE | 2.1.5.RELEASE | 2.1.8.BUILD-SNAPSHOT     |
| spring-cloud-task         | 1.2.4.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-vault        | 1.1.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-gateway      | 1.0.3.RELEASE  | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-openfeign    |                | 2.1.2.RELEASE | 2.1.3.BUILD-SNAPSHOT     |
| spring-cloud-function     | 1.0.2.RELEASE  | 2.0.2.RELEASE | 2.0.3.BUILD-SNAPSHOT     |

### [SpringCloud 全家桶](https://spring.io/projects/spring-cloud)

#### spring-cloud-netflix

##### eureka 原理：

- 服务启动时，调用 eureka 接口进行注册，Eureka Server 会存储这些信息，IP、端口、微服务名称等
- 微服务启动之后，会周期性的发送心跳进行“续租”，默认 30 秒
- 如果一定时间没有“续租”，默认 90 秒，就销毁微服务实例
- 默认情况下，Eureka Server 也是一个 Eureka Client，集群下会互相复制服务注册表中的信息进行同步
- Eureka Client 会缓存注册表的信息，减少Eureka Server 的请求压力，以及容灾

#### spring-cloud-openfeign

#### spring-cloud-zuul 微服务网关





## 项目创建的目的？

第一个目的是为了本人能够熟悉和学习 Spring-Cloud 相关知识，不过在做了一段时间之后还是希望能够分享出来让更多的同学一起学习和讨论。



## 项目结构如何？

- scaffold-business [业务服务提供者](#) 端口从8850 - 8860
    - scaffold-business-sys-service [系统业务微服务-业务模块](#) 端口 8850
    - scaffold-business-job-service [定时任务微服务-业务模块](#) 端口 8851
    - scaffold-business-thirdparty-service [第三方业务微服务-业务模块](#) 端口 8852
- scaffold-business-api [业务API包 用于接口与实现分离](#)
    - scaffold-business-sys-api [系统资源、菜单、权限等API封装](#)
    - scaffold-business-job-api [定时任务API封装](#)
    - scaffold-business-thirdparty-api [第三方服务API封装](#)
- scaffold-core [工具类以及各种公共代码](#)
    - scaffold-core-code [每个模块都会用到的公共代码，Bean，config等](#)
    - scaffold-core-common [工具类模块，公共代码](#)
    - scaffold-core-plugin [自动代码生成插件模块](#)
- scaffold-eureka [注册中心Eureka](#) 端口 8761 - 8771
- scaffold-zuul [网关服务](#) 端口 8861 - 8870
- scaffold-config-server [配置服务端服务](#) 端口 8871 - 8881
- scaffold-config-client [配置客户端服务](#) 端口 8880 - 8891
- scaffold-tx-manager [分布式事务协调服务](#) 端口7970 
- scaffold-feign [Feign模块](#)
    - scaffold-feign-sys [feign-sys模块](#)
    - scaffold-feign-job [feign-job模块](#)
    - scaffold-feign-thirdparty [feign-thirdparty模块](#)
- scaffold-route [主业务消费者](#) 端口从8750 - 8760
    - scaffold-route-operate [后台管理接口及页面](#) 端口 8750
    - scaffold-route-app [APP客户端接口](#) 端口 8751

## 如何快速开始？

### 1. 本地直接启动

- 下载/克隆项目到本地 

    ```java
    git clone https://github.com/Fatezhang/scaffold-cloud
    ```
- 安装MySql数据库并启动
- 创建数据库scaffold_cloud_base 和 tx_manager
- 修改 scaffold-cloud 中微服务的数据库链接配置，本地运行只需要修改application-local.yml
- 安装redis服务并启动，修改scaffold-core-code配置文件中的配置，同样只需要修改local中的
- 安装Rocket MQ服务，同样修改配置
- 如果有需要，注册阿里OSS，并修改配置中的配置
- 启动EurekaApplication注册中心
- 启动TxlcnApplication分布式事务协调服务
- 启动SysServiceApplication，加载数据库字典等配置到缓存、提供后台管理微服务（权限、操作员、角色、国际化配置等）
- 启动RouteOperateApplication服务，默认端口为8750
- 访问http://localhost:8750/ßß
- 默认账号密码为admin/admin123



### 2. 使用 docker 部署

#### docker 启动 : Linux 或者 Mac 下使用如下脚本, Windows 环境自行按照脚本中的示例执行 

#### `mvn clean package docker:build -Pdocker`

    1. 进入项目所在目录
    2. 执行 `./.scripts/recreate-docker-image.sh` 创建 docker 镜像
    3. 执行 `./.scripts/start-docker-service.sh` 即使用 docker-compose 启动
## 项目未来还需要做什么？

- 更改项目注册发现中心，也许用 nacos 或者 zookeeper
- 加入更多 spring-cloud 周边服务，包括各种监控平台等
- CI/CD 使用 Jenkins 或者 BuildKite
- 使用 docker 容器化部署
- ElasticSearch 日志收集
- [xxl-job](https://github.com/xuxueli/xxl-job) 分布式定时任务 https://www.xuxueli.com/xxl-job/
- 整合[第三方开源库](https://github.com/justauth/JustAuth)用以登录、支付等
- 最后，实际开发一些业务功能





---

#### 参考：

> https://juejin.im/post/5c28f2fe51882565a15776fb
>
> https://juejin.im/post/5de740566fb9a0165721b744
>
> https://juejin.im/post/5dc220126fb9a04aa660dcfb

