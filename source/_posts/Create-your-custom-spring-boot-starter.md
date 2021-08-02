---
title: 编写自定义的 SpringBoot starter 实现自动装配
date: 2021-07-30 16:47:15
categories: [SpringBoot]
tags: [ SpringBoot, SpringBoot starter ]
---

## 前言

_记得几年前我在刚开始接触 SpringBoot/SpringCloud，就对SpringBoot 如何实现自动装配产生了很大的好奇。但是当时技术能力尚浅，没能对这一方面了解的很透彻，只是在想如果有朝一日我也能写一个 Starter 提供给别人用就好了。最近我准备写一个 Starter。所以这篇博客就来总结一下，什么是 SpringBoot 自动装配以及如何实现自己的 Starter。_

<!--more-->

## 什么是 **Spring Boot 的 AutoConfiguration**

#### 什么是 SpringBoot 的自动装配

实际上是类似于 SPI(Java Service Provider Interface) 机制， SpringBoot 在启动的时候会扫描 `classpath`下面的这个文件 `META-INF/spring.factories`， 包括所有依赖中的该文件都能够被 SpringBoot 扫描到。然后将文件中配置的类加载到 Spring 容器中，并执行类中定义的操作，比如按需创建更多的 Bean。如下，这是`spring-boot/spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/spring.factories` [🔗](https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/src/main/resources/META-INF/spring.factories#L25)中的片段：

```yml
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.cassandra.CassandraAutoConfiguration,\
org.springframework.boot.autoconfigure.mongo.MongoAutoConfiguration,\
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

该文件中，key 为`org.springframework.boot.autoconfigure.EnableAutoConfiguration`, value 为`org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration`。 SpringBoot 会去扫描该文件并加载RabbitAutoConfiguration 。这就是 SpringBoot 的自动装配机制。

想要更加深入了解`EnableAutoConfiguration`是如何工作的、如何读取加载`spring.factories`，请查看其源码，这里不再详述。

#### 如何实现按需加载

用`RabbitAutoConfiguration`举例

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ RabbitTemplate.class, Channel.class })
@EnableConfigurationProperties(RabbitProperties.class)
@Import(RabbitAnnotationDrivenConfiguration.class)
public class RabbitAutoConfiguration {...}
```

`@ConditionalOnClass`注解标记了，当加载了`RabbitTemplate.class, Channel.class`的时候（也就是说当你的 SpringBoot 项目中引入了 Rabbit 的依赖的时候），才去创建该 bean/configuration`RabbitAutoConfiguration。`

在 SpringBoot 中，有很多[类似的注解](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/condition)，可以实现条件加载 bean 的功能：

```java
@ConditionalOnBean
@ConditionalOnClass
@ConditionalOnCloudPlatform
@ConditionalOnExpression
@ConditionalOnJava
@ConditionalOnJndi
@ConditionalOnMissingBean
@ConditionalOnMissingClass
@ConditionalOnNotWebApplication
@ConditionalOnProperty
@ConditionalOnResource
@ConditionalOnSingleCandidate
@ConditionalOnWarDeployment
@ConditionalOnWebApplication
```

## 实现自己的 SpringBoot Starter

_现在我们了解了SpringBoot 的自动装配和按需加载，已经可以开始尝试写一个自定义的 starter 了。_

#### 首先使用gradle创建一个SpringBoot 项目，引入依赖

```groovy
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter:2.5.3'
  implementation 'org.springframework.boot:spring-boot-starter-web:2.5.3'
  implementation 'org.springframework.boot:spring-boot-autoconfigure:2.5.3'

  implementation 'net.logstash.logback:logstash-logback-encoder:6.+'

  testImplementation 'org.junit.jupiter:junit-jupiter-api:5.5.2'
  testImplementation 'org.junit.jupiter:junit-jupiter-engine:5.5.2'
  testImplementation('org.springframework.boot:spring-boot-starter-test:2.5.3')

  annotationProcessor "org.springframework.boot:spring-boot-configuration-processor:2.5.3"
}
```

其中 ` annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor:2.5.3'` 可以生成你自定义的 Properties 的 Metadata，这样你就可以在引入这个自定义 starter 之后，在`application.properties`中像写其他配置一样写自己的自定义配置。参考[这里]([Spring Boot Reference Guide](https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/reference/htmlsingle/#configuration-metadata-annotation-processor))。[源码查看](https://github.com/Fatezhang/Barrier/blob/master/build.gradle)。

#### 编写一个`spring.factories`文件

在自己的 starter 中编写文件 `src/main/resources/META-INF/spring.factories`

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.barrier.configuration.BarrierAutoConfiguration
```

上面的代码声明了，在 SpringBoot 加载的时候，加载自定义自动配置类`BarrierAutoConfiguration`。

`BarrierAutoConfiguration` 如下：

```java
@Configuration
@ConditionalOnBean(Marker.class)
@EnableConfigurationProperties({BarrierProperties.class})
public class BarrierAutoConfiguration implements WebMvcConfigurer {
  ...
}
```

其中 `@ConditionalOnBean(Marker.class)` 标记了这个 configuration 只有在 bean `Marker` 存在的时候才被加载到 Spring Context 中。那么 Marker 类是什么呢？

```java
@Configuration(proxyBeanMethods = false)
public class EnableBarrierMarkerConfiguration {
    @Bean
    public Marker barrierMarker() {
        return new Marker();
    }
    static class Marker {
        public Marker() {
            log.info("BarrierAutoConfiguration: enableBarrierMarkerBean creating...");
        }
    }
}
```

Marker 类是一个标记类，在`EnableBarrierMarkerConfiguration`中被创建出来，加入到 SpringContext 中去的。那么何时这个 configuration 才会被加载呢？或者说我们如何控制该 configuration 被加载？

#### 创建一个注解实现按需开启 starter

Spring 提供了一个注解 `@Import`，可以提供使用者动态的去加载指定的 bean，尤其是去加载 configuration。

首先你要了解一个前提，SpringBoot 或者 Spring 是无法加载一个外部依赖中的 bean 的。所以我们在自己的 SpringBoot 项目中使用这个 starter 中，在SpringBoot 启动类中这样写：

```java
@Import(EnableBarrierMarkerConfiguration.class)
@SpringBootApplication
public class BootApplication {
    public static void main(String[] args) {
        SpringApplication.run(BootApplication.class, args);
    }
}
```

这样，我们在自己的项目中就能够注入 Marker 这个 bean 了，也就间接地开启了`BarrierAutoConfiguration`。

但是这样写不够优雅，我们可以创建一个注解：

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({EnableBarrierMarkerConfiguration.class})
  public @interface EnableBarrier {
}
```

在注解中使用`@Import`，导入这个 configuration。 

#### 在 SpringBoot 项目中使用

这样，在 SpringBoot 项目中，引入我们自定义的 starter 之后，使用`@EnableBarrier`就能开启我们自己的 starter 的功能了。

```java
@EnableBarrier
@SpringBootApplication
public class BootApplication {
    public static void main(String[] args) {
        SpringApplication.run(BootApplication.class, args);
    }
}
```

---

**最后，自定义 starter 的源码可以看[这里](https://github.com/Fatezhang/Barrier)。**

