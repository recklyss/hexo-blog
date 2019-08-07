---
title: 我项目中使用的分布式事务：LCN
date: 2019-08-03 11:14:44
categories: [分布式,分布式事务]
tags: [分布式,分布式事务,数据库,微服务]
---

![官网首页](tx.png)

<!--more-->

<a href="#### 其他关于分布式事务的总结整理">其他关于分布式事务的总结整理</a>

#### 背景

由于公司项目是使用dubbo进行开发的分布式服务，所以项目中有很多涉及到分布式事务问题的场景。比如有两个模块：用户模块和账户资金模块。有一个场景是用户被邀请成为系统的新用户，需要先初始化用户信息，然后再去账户资金模块初始化用户账户信息。两个不同的模块为两个不同的RPC服务，分别被调用然后插入数据，这时候如果账户资金插入失败，不加入分布式事务的话用户直接初始化成功。我们希望这种情况下用户插入的信息被回滚，所以需要引入分布式事务来进行业务处理。

#### 使用的框架

经过调研，我们发现TX-manager这个服务比较适合我们的业务场景，我们打算引入并使用LCN事务模式来进行服务中的分布式事务的业务处理。

在[官网](https://www.txlcn.org/zh-cn/index.html)下载对应的服务，并引入项目或者单独启动：

![官网首页](tx.png)

引入依赖：

```xml
<dependency>
    <groupId>com.codingapi</groupId>
    <artifactId>transaction-dubbo</artifactId>
    <version>${lcn.version}</version>
</dependency>
<dependency>
    <groupId>com.codingapi</groupId>
    <artifactId>tx-plugins-db</artifactId>
    <version>${lcn.version}</version>
</dependency>
```

使用：

在服务的发起方使用注解`@TxTransaction(isStart = true)`

```java
@Override
@TxTransaction(isStart = true)
public ExperienceLogVO doUseExperience(Long userId, Long experienceRecordId, ExperienceLogCreateModel createModel) {
 	// ... do something  ...
    userFacade.insert();
 	// ... do something  ...
    accountFacede.insert();
}
```

在服务的参与方使用注解`@TxTransaction`标识即可

```java
@TxTransaction
public int insert(){
 	// ... do something  ...
}
```

然后再启动项目之前，先启动tx-manager服务，作为协调者的角色存在，然后启动项目，调用接口的时候就可以使用分布式事务了。

#### 其他关于分布式事务的总结整理

