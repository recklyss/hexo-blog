---
title: spring-boot微服务搭建
date: 2018-03-27 17:01:21
categories: [springboot]
tags: [springboot]
---

[本人的csdn传送门](http://blog.csdn.net/qq_26627671/article/details/76563127)
### 前言
> 进行web开发的时候Java程序员们难免会碰到那种很小的服务，比如就提供一个生成订单号的接口，或者一个上传文件的服务。而这时我们再去使用SpringMVC这种体量稍大、配置繁琐的框架开发难免会加大工作量，而且是不必要的。这个时候我们就可以选择使用这个微服务框架——springboot进行开发。

Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，从而使开发人员不再需要定义样板化的配置。通过这种方式，Spring Boot致力于在蓬勃发展的快速应用开发领域(rapid application development)成为领导者。

<!--more-->

----------
### springboot框架的搭建与简单的REST风格的MVC架构demo
#### 首先，建立一个新的maven工程，pom文件主要内容如下：
```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.2.1.RELEASE</version>
    <relativePath/>
  </parent>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <java.version>1.8</java.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
```
其中核心依赖是	`spring-boot-starter-web`
```
	<dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
```
访问静态资源文件可以加入模板：
```
	<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
    </dependency>
```
#### 编写Application.java文件，存放于src/main/java这个目录下
##### 这里是springboot的核心启动类
```
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
@Configuration
@ComponentScan
@EnableAutoConfiguration
public class Application{
	
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```
事实上，这个时候已经把框架搭建好了，运行以上main方法即可启动这个项目，但是我们现在看不到效果，接下来，就可以像SpringMVC一样加入MVC三层结构的代码了，目录结构如下图：

![REST风格的MVC架构demo项目目录结构](http://upload-images.jianshu.io/upload_images/3327380-fb0da321cb7b5abb?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中各层代码如下：
#### controller
```
package com.zhang.controller;
import java.util.HashMap;
import java.util.Map;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.servlet.ModelAndView;
import com.zhang.entity.Photo;
import com.zhang.service.mainService;
@RestController
@RequestMapping("/photo")
public class mainController {

	@Autowired
	private mainService mainservice;
	@RequestMapping("/")
	public ModelAndView index(ModelAndView mav){
		mav.addObject("hello", "这是项目主页，访问根目录到达~~");
		mav.setViewName("index");
		return mav;
	}
	@RequestMapping("/getPhoto")
	public Object doIt(){
		Map<String, Photo> map = new HashMap<String, Photo>();
		map.put("photo", mainservice.getPhotoById(123));
		return map;
	}
}

```
#### service实现类
```
package com.zhang.service.impl;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.zhang.dao.mainDao;
import com.zhang.entity.Photo;
import com.zhang.service.mainService;
@Service("mainservice")
public class mainServiceImpl implements mainService {

	@Autowired
	private mainDao maindao;
	@Override
	public Photo getPhotoById(int id) {
		return maindao.getPhotoNameById(id);
	}

}

```
#### dao实现类
```
package com.zhang.dao.impl;

import org.springframework.stereotype.Repository;

import com.zhang.dao.mainDao;
import com.zhang.entity.Photo;

@Repository("maindao")
public class mainDaoImpl implements mainDao {

	@Override
	public Photo getPhotoNameById(int id) {
		Photo p = new Photo();
		p.setId(123);
		p.setName("雪山行纪念照");
		return p;
	}

}

```
#### 实体类photo
```
package com.zhang.entity;
public class Photo {
	private int id;
	private String name;
	public int getId() {
		return id;
	}
	public void setId(int id) {
		this.id = id;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
}

```
#### 启动项目

![项目启动日志](http://upload-images.jianshu.io/upload_images/3327380-8ed38aa7e3e43e40?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 项目默认端口为8080，在浏览器中访问刚才的controller会看到：

![访问结果](http://upload-images.jianshu.io/upload_images/3327380-75812c968f3ff846?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


----------
### 如上，一个REST风格的MVC架构的demo项目就搭建完成了。
