---
title: dubbo服务降级
date: 2019-04-12 11:48:06
categories: [dubbo]
tags: [springboot,dubbo,分布式服务降级]
---
### 前言

&nbsp;&nbsp;&nbsp;&nbsp;在分布式服务或者一些微服务当中，经常性的出现各个服务之间相互调用，业务处理流程耦合起来的情况。比如当用户在平台下单时，我们需要给用户生成订单之后发站内信或者短信通知用户订单生成成功。那么很多时候代码的编写就会是：
&nbsp;&nbsp;&nbsp;&nbsp;`调用订单模块生成订单->调用短信模块通知用户->调用其他模块处理更多业务逻辑`
&nbsp;&nbsp;&nbsp;&nbsp;可是当我们无足轻重的一个短信通知模块挂掉或者报错的时候，我们当然不希望整个业务逻辑就这样停止。那么这个时候，就需要引入服务降级的机制，为整个业务逻辑进行解耦合。

&nbsp;&nbsp;&nbsp;&nbsp;使用服务降级可以防止我们服务中间不影响整体流程的模块出错导致整个业务处理雪崩。将核心业务保证完整性，非核心业务弱化。
<!--more-->
*<font style="color: red">本文使用  `springboot+dubbo` 进行服务降级的演示</font>*

### dubbo自带的mock进行服务降级，也叫本地伪装
##### dubbo作为阿里巴巴开源的最流行的服务治理框架，在提供了远程调用的同时也提供了服务降级的功能。
具体使用

dubbo mock的使用非常简单，即在我们平时进行开发时，编写impl实现类实现接口作为服务提供者的同时，编写mock实现类并覆盖所有接口中的方法。
官方更详细的文档[戳这里](http://dubbo.apache.org/zh-cn/docs/user/demos/local-mock.html)

比如有接口：
```
public interface SysOperateFacade {
    /**
     * 根据用户名查询操作员信息
     */
    SysOperateVO findByUserName(String username);
}
```
在实现类进行相应操作
```
@Service //这里Service是dubbo的注解
public class SysOperateFacadeImpl implements SysOperateFacade {
  @Resource
  SysOperateService sysOperateService;

  @Override
  public SysOperateVO findByUserName(String username) {
      return sysOperateService.findByUserName(username);
  }
}
```
编写mock实现类覆盖findByUserName方法 注意 mock的类名必须是 接口名+Mock
```
public class SysOperateFacadeMock implements SysOperateFacade {
    @Override
    public SysOperateVO findByUserName(String username) {
        System.out.println("调用到dubbo mock 的findByUserName方法。。。。。。。");
        return new SysOperateVO();
    }
}
```
最后，在调用的地方加上注解`@Reference(mock = "true")`进行使用即可
```
@Controller
@RequestMapping("/sys/sysOperate")
public class SysOperateController extends BaseController {

    @Reference(mock = "true")
    private SysOperateFacade sysOperateFacade;

    @ResponseBody
    @RequestMapping("/test")
    public SysOperateVO test(String username){
        return sysOperateFacade.findByUserName(username);
    }
}
```

### 使用 spring cloud Hystrix进行服务降级

在服务调用方模块加入依赖
```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
    <version>1.4.6.RELEASE</version>
</dependency>
```
如果出现以下报错也许是由于Springboot与这个依赖版本不对应，修改下版本
```
java.lang.NoSuchMethodError: org.springframework.boot.builder.SpringApplicationBuilder.<init>([Ljava/lang/Class;)V at org.springframework.cloud.bootstrap.BootstrapApplicationListener.bootstrapServiceContext(BootstrapApplicationListener.java:170) at org.springframework.cloud.bootstrap.BootstrapApplicationListener.onApplicationEvent(BootstrapApplicationListener.java:104) at org.springframework.cloud.bootstrap.BootstrapApplicationListener.onApplicationEvent(BootstrapApplicationListener.java:70) at org.springframework.context.event.SimpleApplicationEventMulticaster.doInvokeListener(SimpleApplicationEventMulticaster.java:172) at org.springframework.context.event.SimpleApplicationEventMulticaster.invokeListener(SimpleApplicationEventMulticaster.java:165) at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:139) at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:122) at org.springframework.boot.context.event.EventPublishingRunListener.environmentPrepared(EventPublishingRunListener.java:74) at org.springframework.boot.SpringApplicationRunListeners.environmentPrepared(SpringApplicationRunListeners.java:54) at org.springframework.boot.SpringApplication.prepareEnvironment(SpringApplication.java:325) at org.springframework.boot.SpringApplication.run(SpringApplication.java:296) at org.springframework.boot.SpringApplication.run(SpringApplication.java:1118) at org.springframework.boot.SpringApplication.run(SpringApplication.java:1107) at com.gjj.p2p.BasicsApplication.main(BasicsApplication.java:18)
```
#####  具体使用`hystrix`进行服务降级

`hystrix`的使用也是非常简单，只需要在服务调用方即消费者方springboot启动类上加上注解 `@EnableHystrix`

然后使用如下方式，指定服务出错或者熔断后调用的方法
```
@ResponseBody
@RequestMapping("/test")
@HystrixCommand(fallbackMethod = "fallback")
public String test(String message){
    return sysMenuFacade.test(message);
}

public String fallback(String message){
    return "sysMenuFacade挂了 调用到fallback " + message;
}
```
这样当出现问题之后就会调用得到fallback方法
还可以在这个controller上直接指定注解`@DefaultProperties(defaultFallback = "fallback")`以免编写大量重复代码

### 总结

服务降级与熔断机制在我们实际生产以及日常开发中都是是非常有必要使用的，例如我们在日常开发中，需要调用别人的模块，但是又不是非常依赖这个模块的数据，我们可以使用以上的方式构造“假的”调用结果。这样就不用为了调试某行代码去启动大量的服务了。

最后针对dubbo的mock机制以及`hystrix`，我觉得`hystrix`更像是try{}catch{}。
