---
title: How to use Junit5 to test you java application elegantly
date: 2020-01-15 14:25:14
categories: [Junit5, Unit Test]
tags: [Junit5, Java 基础, 单元测试, unit test]
---

<h1 align="center">Junit5 的一些实际开发中常用的功能【 TDD 向 】</h1>

<a href="/ppt/junit-study.html"><h2 align="center">点击阅读PPT</h2></a>

![Junit](junit5.png)

<!--more-->

> 这篇博客本来应该写于鼠年过年之前，谁想到犯了个懒，现在怀着沉重的心情写下这篇博客~
>
> 因为中国现在正在受着新型冠状病毒疫情的危害，大过年的门都出不去，都躲在家里生怕被感染。无聊的我只能学习学习来打发一下枯燥的假期生活哈哈~

### Junit5

说到 JUNIT5 可能很多人都不陌生，就是一个测试框架而已。但是相信大部分的程序员都没有用过，感觉国内好多公司都没有写测试的习惯，也许是产品催需求催的太紧ε=(´ο｀*)))。我就不在这里介绍 Junit5 了，没有前置知识的话需要看一下：[Junit5](https://junit.org/junit5/)，我只在这篇博客介绍一下我在工作中是如何使用 Junit5 的。





### 基本的测试代码

首先 gradle需要引入 Junit5 以及其他相关的依赖：

```groovy
testImplementation 'org.mockito:mockito-core:2.24.0'
testImplementation 'org.junit.jupiter:junit-jupiter-api:5.2.0'
testImplementation 'org.junit.jupiter:junit-jupiter-params:5.2.0'
testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.2.0'
testImplementation "org.testcontainers:junit-jupiter:1.12.3"
testImplementation "org.testcontainers:postgresql:1.12.3"
testImplementation 'org.mockito:mockito-junit-jupiter:3.1.0'
testImplementation 'org.hamcrest:hamcrest:2.2'
```

然后编写一个简单的测试用例：

```java
@Test
void shouldReturnObjectsGivenValidIdWhenGetObjects() {
  when(objectService.findApplications(OBJECT_ID)).thenReturn(mockObjects);

  var responseEntity = objectsController.getObjects(null);
  
  assertThat(HttpStatus.OK).isEqualTo(responseEntity.getStatusCode());
  assertThat(responseEntity.getBody()).isEqualTo(mockGetObjectsResponse);
}
```

说明一下，以上测试用例的编写需要注意以下几点：

1. 方法命名
   一般的写测试的时候，方法的命名要求解释测试的功能，需要包含 should/given/when 条件。如以上方法的命名`shouldReturnObjectsGivenValidIdWhenGetObjects`是指当调用getObjects()的时候，给定一个合法的 ID，能够返回一组 Objects。
2. 方法的条件、执行、以及对于结果的验证的代码换行隔开
3. 使用 assertThat()验证结果：
   `assertThat`方法是包`import static org.assertj.core.api.Assertions.assertThat;`中的方法。它接收一个实际的值，然后再使用`isEqualTo`等方法判断上一个执行结果是否正确。

### 在类中的测试执行之前执行一次代码

以下代码使用`@BeforeAll`注解在方法 setupMDC 上面，并且这个方法必须是 static 静态方法。是指在所有类中的测试方法执行之前只执行一次该方法。一般用于设置无状态的全局变量。

```
@BeforeAll
static void setupMDC() {
    MDC.put(TRANSACTION_ID, UUID.randomUUID().toString());
}
```

### 在每个测试方法之前都执行一些代码

以下代码使用`@BeforeEach`注解，可以在每一个方法执行之前都执行该方法，用于每个方法执行前的初始化或者做一些共同的 mock 操作，相当于 AOP 的 Before。

```java
@BeforeEach
void setup() {
  when(service.save(any(String.class), any(UUID.class), any(UUID.class))).thenReturn(null);
}
```

### 其他

其他的比如`@AfterAll` `@AfterEach`等等功能依次类推。

### 测试分层

在使用 Junit5 测试过程中，最让我觉得方便的是对于测试类的结构划分方式：内部类。通过内部类，我们可以将我们要测试的东西使用类结构的形式去进行描述（使用注解`@Nested`修饰），然后再在类中编写相应的测试方法进行具体的测试。

比如一个 Controller 需要测试 create/update/get 等方法，就可以将这几个方法依次编写内部类分开描述，然后再在类中对于不同的分支编写测试方法进行单元测试，如下：

```java
@ExtendWith(MockitoExtension.class)
class ObjectsControllerTest {
  
    @Nested
    class CreateObjects {
      @Test
      void someTest(){
      	// some code
      }
    }
  
    @Nested
    class UpdateObjects {
      @Test
      void someTest(){
      	// some code
      }
    }
  
    @Nested
    class GetObjects {
      @Test
      void someTest(){
      	// some code
      }
    }
}
```

### 总结

以上就是我在编写 unit 测试的时候的一些小小的总结，以后会继续加深对于测试代码编写的学习，实际上测试才是写代码过程中最重要的一环，可以保障系统的功能的正确性，还能保护重构。Junit5 为 JAVA 程序员提供了更强大、方便的测试框架，值得深入研究使用。