---
title: ' 解决@ResuestBody中的 JSON 自动转化非 boolean 为 boolean 值'
date: 2019-12-17 11:39:58
categories: [SpringBoot]
tags: [Spring,SpringMVC,SpringBoot,JSON,converter,RequestBody]
---

![封面](code.jpg)

<!--more-->

## 问题出现

当我们在 SpringBoot 中写 API 的时候，通常我们会使用`@RequestBody`注解一个参数将这个对象标记之后，然后我们在请求头使用`application/json`调用这个 API，传入 JSON 的 body 体，就可以自动的将我们的 JSON 转化成 JAVA 对象。但是，当我使用的 JAVA 对象中有个 Boolean 的字段的时候，我的 JSON 的 body 对这个对象传数值、”True”等等其他值得时候，往往会被默认转成相应的 true 或者 false。例如传入{“able”:0}的时候，我的对象中的 able 字段就是 false。但是我不想要这个功能，我希望接口调用者传的类型都是 Boolean 类型。

## Debug 源码

首先你要知道，在 SpringBoot 或者 SpringMVC 中对于request 和 response 的处理是使用的消息转换器处理的。所以我在 debug 源码的时候发现，SpringBoot 使用`MappingJackson2HttpMessageConverter`处理 JSON 转化成对象，然后实际的转化方法`MappingJackson2HttpMessageConverter`没有重写，而是交给父类`AbstractJackson2HttpMessageConverter`的方法，在该类的第239 行会发现实际上是使用的 `this.objectMapper.readValue(inputMessage.getBody(), javaType);` 将一个 JSON 字符串转化成 JAVA 对象。再进去看下：

```java
public <T> T readValue(InputStream src, JavaType valueType)
  throws IOException, JsonParseException, JsonMappingException
{
  return (T) _readMapAndClose(_jsonFactory.createParser(src), valueType);
} 
```

而`_readMapAndClose`方法是这样的(重点看下 4013 行)：

```java
protected Object _readMapAndClose(JsonParser p0, JavaType valueType)
  throws IOException
{
  try (JsonParser p = p0) {
    Object result;
    JsonToken t = _initForReading(p, valueType);
    final DeserializationConfig cfg = getDeserializationConfig();
    final DeserializationContext ctxt = createDeserializationContext(p, cfg);
    if (t == JsonToken.VALUE_NULL) {
      // Ask JsonDeserializer what 'null value' to use:
      result = _findRootDeserializer(ctxt, valueType).getNullValue(ctxt);
    } else if (t == JsonToken.END_ARRAY || t == JsonToken.END_OBJECT) {
      result = null;
    } else {
      JsonDeserializer<Object> deser = _findRootDeserializer(ctxt, valueType);
      if (cfg.useRootWrapping()) {
        result = _unwrapAndDeserialize(p, ctxt, cfg, valueType, deser);
      } else {
        result = deser.deserialize(p, ctxt);
      }
      ctxt.checkUnresolvedObjectId();
    }
    if (cfg.isEnabled(DeserializationFeature.FAIL_ON_TRAILING_TOKENS)) {
      _verifyNoTrailingTokens(p, ctxt, valueType);
    }
    return result;
  }
}
```

这行代码`result = deser.deserialize(p, ctxt);`使用一个反序列化对象进行 JSON 的反序列化，这里如果传入的是数字转化成 Boolean的话就是用的是`NumberDeserializers`中的`BooleanDeserializer`，而它的``方法是这样的：

```java
@Override
public Boolean deserialize(JsonParser p, DeserializationContext ctxt) throws IOException
{
  JsonToken t = p.getCurrentToken();
  if (t == JsonToken.VALUE_TRUE) {
    return Boolean.TRUE;
  }
  if (t == JsonToken.VALUE_FALSE) {
    return Boolean.FALSE;
  }
  return _parseBoolean(p, ctxt);
}
```

然后`_parseBoolean`中将其他的数值转化成 Boolean。

## 解决方式

以上就是整个 debug 的全过程了，总结一下就是`AbstractJackson2HttpMessageConverter`中会默认地将非 Boolean 的数值转化成 Boolean。那么如何解决这个问题呢？

首先使用搜索引擎解决😹然后搜索不到。。。然后我在 StackOverFlow 上提了一个[问题 <click](https://stackoverflow.com/questions/59353379/springboot-atomically-convert-integer-to-boolean-with-requestbody-annotation-h/59355180#59355180)。简单来说就是以下这种方式：

- 自定义自己的反序列化工具，然后让 Spring 去管理这个配置类

  ```java
  @Configuration
  public class SystemConfiguration {
  
      @Bean
      public SimpleModule addDeserializer() {
          return new SimpleModule().addDeserializer(Boolean.class, new MyDeserializer());
      }
  
      static class MyDeserializer extends JsonDeserializer<Boolean> {
          @Override
          public Boolean deserialize(JsonParser p, DeserializationContext ctxt) throws IOException {
              JsonToken t = p.getCurrentToken();
              if (t == JsonToken.VALUE_TRUE) {
                  return Boolean.TRUE;
              }
              if (t == JsonToken.VALUE_FALSE) {
                  return Boolean.FALSE;
              }
              if (t == JsonToken.VALUE_STRING) {
                  String value = p.getValueAsString();
                  return value.equals("true") ? true : value.equals("false") ? false : null;
              }
              return null;
          }
      }
  }
  ```

这样就将所有 API 中，存在 Boolean 的情况都处理掉了。只能接受 Boolean 值或者字符串的”true”或者”false”。

但是作为一个优秀(pa ma fan)的👩‍💻coder😷，我们应该保证自己修改的代码不会影响到其他人或者其他模块，随便的修改全局的配置不太好。所以我寻找到了一个更加୧(๑•̀◡•́๑)૭的方式 —— 编写一个该属性的 set 方法即可。

因为Deserializer会将读取到的 JSON 的值通过 set 方法填入对象中，所以这种方式也是完全可行的，如下：

```java
public void setAble(Object value) {
  if (value instanceof Boolean) {
    able = (Boolean) value;
  }
  if ("true".equals(value)) {
    able = true;
  }
}
```

至此，大功告成。但是实际上整个过程 debug 的时候是最有意思的，可以了解到它在转换的过程中实际上都做了些什么事情。