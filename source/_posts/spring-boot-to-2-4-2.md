---
title: 由 SpringBoot 升级到 2.4.2 引发的 Jackson 的 JsonFormat 问题排查
date: 2021-01-20 19:37:43
categories: [springboot]
tags: [springboot, DateTimeFormatter, jackson, lenient, JsonFormat]
---

![3hmYwN](https://cdn.jsdelivr.net/gh/Fatezhang/FigureCloud@master/uPic/3hmYwN.jpg)


<!--more-->



## 事情是这样的

在不久的以前，我们项目的 Tech Lead 决定在git repo中引入 DependaBot 来对项目中的依赖做检查并升级。我们的一个使用 SpringBoot 的服务也就这样成了待升级依赖的一份子。我们待升级的依赖包括但不限于：

- Bump newrelic-agent from 5.8.0 to 6.3.0 …
- Bump guava from 28.0-jre to 30.1-jre …
- Bump spring-hateoas from 1.1.0.RELEASE to 1.2.3 …
- Bump postgresql from 42.2.8 to 42.2.18 …
- Bump cloudwatch from 2.13.41 to 2.15.66 …
- Bump json-schema-validator from 4.2.0 to 4.3.3 …
- Bump org.springframework.boot from 2.2.5.RELEASE to 2.4.2 …
- Bump io.spring.dependency-management …
- Bump io.freefair.lombok from 4.1.3 to 5.3.0 …
- Bump org.flywaydb.flyway from 6.1.3 to 7.5.0 …

可以看到，几乎都将这些依赖升级到了最新的版本，甚至 SpringBoot2.4.2 是在这次升级的前三天 release 的。但是我们不慌，升级依赖什么的对我们来说跟喝水一样简单，因为... 

```groovy
jacocoTestCoverageVerification {
  dependsOn 'jacocoTestReport'
  violationRules {
    rule {
      element = 'CLASS'
      limit {
        minimum = 1.0
      }
    }
  }
}
```

我们的代码的测试覆盖率的要求是惊人的100%🤣 这在我之前的公司是绝对无法实现的。不仅仅是 unit test， 我们还有 integration 测试覆盖，还有用到  cypress 又一次覆盖了所有的 endpoint。不就是改改代码么/升级依赖啥的么，随便玩。

## 于是

梭哈！👨🏻‍💻👨🏻‍💻👨🏻‍💻👨🏻‍💻👨🏻‍💻👨🏻‍💻👨🏻‍💻升级，跑测试！

几分钟后：![do0JqI](https://cdn.jsdelivr.net/gh/Fatezhang/FigureCloud@master/uPic/do0JqI.png)

行嘛，不出我所料（才怪🙃）果然挂了。

打开log一看， emmm... 

```bash
Resolved [org.springframework.http.converter.HttpMessageNotReadableException: JSON parse error: Cannot deserialize value of type `java.time.LocalDate` from String "2020-01-15": Failed to deserialize java.time.LocalDate: (java.time.format.DateTimeParseException) Text '2020-01-15' could not be parsed: Unable to obtain LocalDate from TemporalAccessor: {YearOfEra=2020, MonthOfYear=1, DayOfMonth=15},ISO of type java.time.format.Parsed; nested exception is com.fasterxml.jackson.databind.exc.InvalidFormatException: Cannot deserialize value of type `java.time.LocalDate` from String "2020-01-15": Failed to deserialize java.time.LocalDate: (java.time.format.DateTimeParseException) Text '2020-01-15' could not be parsed: Unable to obtain LocalDate from TemporalAccessor: {YearOfEra=2020, MonthOfYear=1, DayOfMonth=15},ISO of type java.time.format.Parsed
 at [Source: (PushbackInputStream); line: 2, column: 18] (through reference chain: com.example.demo.Demo["localDate"])]
```

汪的发！？😢 用的好好的`@JsonFormat`怎么就突然不好使了？

## 问题排查

#### 打开代码看一下：

我有这么一个对象：

```java
public class Demo {
    public Demo(){}
    public Demo(LocalDate localDate){
        this.localDate = localDate;
    }
    @JsonFormat(pattern = "yyyy-MM-dd", lenient = OptBoolean.FALSE)
    private LocalDate localDate;
    public LocalDate getLocalDate() {
        return localDate;
    }
    public void setLocalDate(LocalDate localDate) {
        this.localDate = localDate;
    }
}
```

其中配置了 localDate 的反序列化为严格模式`lenient = OptBoolean.FALSE`，防止将 number 反序列化为日期，那样是不正确的。

有这么一个 controller：

```java
@RestController
public class Controller {
    @GetMapping("/demo")
    public ResponseEntity<String> test(@RequestBody Demo demo) {
        var localDate = demo.getLocalDate().toString();
        return ResponseEntity.ok(localDate);
    }
}
```

代码很简单，就是有一个对象，接收一个 LocalDate 的属性，用pattern `yyyy-MM-dd` 接收类似于`2020-01-15`这样格式的日期。

但是之前用得好好的升级了 SpringBoot2.4.2之后却用不了了？emmmm... 一定是 SpringBoot 升级升了啥不该升的玩应，🧐我要去 SpringBoot 的升级日志里看看，是不是升级了 Jackson 啥的，万一找到一个大霸哥🦟，提个 PR 不就从此成为顶级开源项目的 contributor 了。。。😎

#### SpringBoot 2.4.2 升级日志

去 GitHub 上打开  [SpringBoot Release v2.4.2](https://github.com/spring-projects/spring-boot/releases/tag/v2.4.2) ， 浏览下 **Bug Fixes** 、 **Documentation**、**Dependency Upgrades**， 发现一行：

> Upgrade to Jackson Bom 2.11.4 #24726

果然，升级了 Jackson 到`2.11.4`。 对比了一下发现我原先的 SpringBoot 中的 Jackson 版本是`2.10.2`， emm... 一般这种稍大的版本升级都伴随着很多 magic 的事情。总之接下来要去 Jackson 的升级日志里面看一下，有什么升级跨越了`2.10.*`和`2.11.*`这两个版本。

#### Jackson 2.11升级日志

这个升级日志在它 GitHub 的 wiki 里，点击[Jackson Release 2.11](https://github.com/FasterXML/jackson/wiki/Jackson-Release-2.11)。 

阅读一下，第一遍竟然没有找到任何线索，阿西吧🥵，通篇与`@JsonFormat`的字眼几乎没有。但是，功夫不负有心人，由于我这个错误是时间类型的转换问题，在如下所示的更改中，发现对于`Java 8date/time`有相关升级：

![xCxT2g](https://cdn.jsdelivr.net/gh/Fatezhang/FigureCloud@master/uPic/xCxT2g.png)

> - [#148](https://github.com/FasterXML/jackson-modules-java8/issues/148): Allow strict `LocalDate` parsing

打开这个 [issue](https://github.com/FasterXML/jackson-modules-java8/issues/148) 看一下，如他们所讨论的，在之前配置了`@JsonFormat(pattern = "yyyy-MM-dd", lenient = OptBoolean.FALSE)`, Jackson 创建的`DateTimeFormatter`还是会使用`ResolverStyle.SMART` smart 模式，并不能阻止非法日期`2019-11-31`的输入。 所以在`2.11`版本之后， 如果设置了`lenient = OptBoolean.FALSE`, `DateTimeFormatter`会使用严格模式，看看代码：

在Jackson 中的`JSR310DateTimeDeserializerBase`这个类中，有这么一个方法`createContextual`， 有这么一段代码：

```java
if (!deser.isLenient()) {
  df = df.withResolverStyle(ResolverStyle.STRICT);
}
```

可是，为什么`DateTimeFormatter`使用了严格模式，会导致上述报错呢？

#### Java8 之后的 java.time 之 DateTimeFormatter

**严格模式下的字符串转LocalDate**

***举个🌰👀👀***

```java
public static void main(String[] args) {
  DateTimeFormatter formatter = DateTimeFormatter
    .ofPattern("yyyy-MM-dd")
    .withResolverStyle(ResolverStyle.STRICT);

  LocalDate localDate = LocalDate.parse("2021-01-20", formatter);
  System.out.println("localDate = " + localDate);
}
```

执行，并抛出异常，转换失败！

```bash
Exception in thread "main" java.time.format.DateTimeParseException: Text '2021-01-20' could not be parsed: Unable to obtain LocalDate from TemporalAccessor: {YearOfEra=2021, DayOfMonth=20, MonthOfYear=1},ISO of type java.time.format.Parsed
```

关键字`YearOfEra`？🧐啊，带年代的年？沃德发😱？

打开类`DateTimeFormatter`搜索一下`yyyy`，发现一段注释里面`y: year-of-era`：

```tex
 * All letters 'A' to 'Z' and 'a' to 'z' are reserved as pattern letters. The
 * following pattern letters are defined:
 * <table class="striped">
 * <caption>Pattern Letters and Symbols</caption>
 * <thead>
 *  <tr><th scope="col">Symbol</th>   <th scope="col">Meaning</th>         <th scope="col">Presentation</th> <th scope="col">Examples</th>
 * </thead>
 * <tbody>
 *   <tr><th scope="row">G</th>       <td>era</td>                         <td>text</td>              <td>AD; Anno Domini; A</td>
 *   <tr><th scope="row">u</th>       <td>year</td>                        <td>year</td>              <td>2004; 04</td>
 *   <tr><th scope="row">y</th>       <td>year-of-era</td>                 <td>year</td>              <td>2004; 04</td>

```

原来，`u`才是代表年的那个字母，而`y`是指带有纪元（era）的年，在`DateTimeFormatter`严格模式下使用，`yyyy-MM-dd`并不合法，正确的使用姿势是`uuuu-MM-dd`！！！

所以`yyyy`要怎么用呢？如下，带上`G`表示一下公元前或者公元后吧。`AD/BC`

```java
DateTimeFormatter formatter = DateTimeFormatter
  .ofPattern("yyyy-MM-dd G")
  .withResolverStyle(ResolverStyle.STRICT);

LocalDate localDate = LocalDate.parse("2021-01-20 AD", formatter);
```

**至此，大功告成，问题解决，依赖也成功升级**

#### 总结一下

问题解决了心情很好，但是反思一下，Java8 都出来这么久了，新的日期时间也用了很多，但是就是忽略了`y`和`u`这么不起眼的小问题！

在问题的排查中，实际上并不如上述流程这样顺利，我还在 Jackson 的 GitHub 里面提了 issue

https://github.com/FasterXML/jackson-modules-java8/issues/199

在我排查 Jackson 的源码的时候，发现他们对于这段代码`df = df.withResolverStyle(ResolverStyle.STRICT);`的升级，并没有很完善的测试。在他们的源码中可以看到test case 都是只是测试了异常情况，并没有覆盖原先本应该正确的 case（可见 unit test 是多么的重要），他们的测试源码如下：

```java
public class LocalDateDeserTest extends ModuleTestBase {
    private final ObjectMapper MAPPER = newMapper();

    final static class StrictWrapper {
        @JsonFormat(pattern="yyyy-MM-dd",
                lenient = OptBoolean.FALSE)
        public LocalDate value;

        public StrictWrapper() { }
        public StrictWrapper(LocalDate v) { value = v; }
    }

    @Test(expected = InvalidFormatException.class)
    public void testStrictCustomFormat() throws Exception
    {
        /*StrictWrapper w =*/ MAPPER.readValue("{\"value\":\"2019-11-31\"}", StrictWrapper.class);
    }
}
```

这个测试的问题在于，将`{ "value" : "2019-11-31"}`改成合法的也能跑过。因为严格模式下， `yyyy-MM-dd`并不合法，同样会跑出`InvalidFormatException`异常。所以我在 Jackson 的`jackson-modules-java8`这个 repo 下还提了一个 PR 去修改他们的测试用例：

https://github.com/FasterXML/jackson-modules-java8/pull/201

 不过也只是简单覆盖一下这个 case，对于其他用到`yyyy`的测试并未做修改，希望我的 PR 能被合进去吧哈哈😜虽然只是单元测试并不是代码功能，但也很有用啊。



