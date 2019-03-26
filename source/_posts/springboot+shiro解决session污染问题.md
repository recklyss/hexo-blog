---
title: springboot+shiro解决session污染的问题
date: 2019-03-09 15:26:49
categories: [Java基础]
tags: [springboot,shiro,session污染]
---

#### 同一个服务器启动多个web项目造成session污染

昨天在敲代码的时候遇到了一个问题，同一个项目，我创建了两个分支，分别使用不同的端口。
但是在测试环境启动的时候我发现，在同一个浏览器上，我只能登陆其中的一个后台。在登陆另一个后台之后，前面那个
又需要再重新登陆了。

原因找了好久，最后F12控制台查看session发现，这两个web项目，使用的都是JSessionId作为cookie的key，在登陆另一个时，浏览器的这个cookie值就会被改变，所以前者就需要在重新登陆了。

<!--more-->
#### 解决方法
在springboot中，对shiro配置进行更改session保存时的cookie的key名称，如下。
```
@Bean
public DefaultWebSessionManager sessionManager() {
    DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
    Cookie cookie = sessionManager.getSessionIdCookie();
    cookie.setName("MySessionId");
    return sessionManager;
}
```
然后在`securityManager`中将我们的`sessionManager`注入进去。
```
/**
 * SecurityManager，权限管理，这个类组合了登陆，登出，权限，session的处理，是个比较重要的类。
 */
@Bean
public DefaultWebSecurityManager securityManager() {
    DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
    securityManager.setRealm(myShiroRealm());
    securityManager.setSessionManager(sessionManager());
    return securityManager;
}
```
只需要这样修改好就可以了。然后重启项目，就会发现，两个web项目都可以同时登陆了。
