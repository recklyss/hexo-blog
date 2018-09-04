---
title: hexo博客文章置顶方式
date: 2018-09-04 19:51:53
tags: [开发日记,博客设置]
---

### 博文置顶
#### 目前已经有修改后支持置顶的仓库，可以直接用以下命令安装
```
npm uninstall hexo-generator-index --save
npm install hexo-generator-index-pin-top --save
```
<!--more-->
#### 然后在需要置顶的文章的Front-matter中加上top: true即可。比如下面这篇文章：
```
---
title: hexo博客置顶
date: 2017-09-08 12:00:25
categories: 博客搭建系列
top: true
---
```
#### 到目前为止，置顶功能已经可以实现了。下面可以设置明确的置顶标志：
##### 打开：/blog/themes/next/layout/_macro 目录下的post.swig文件，定位到`<div class="post-meta">`标签下，紧接着下一行插入如下代码：
```
          {% if post.top %}
            <i class="fa fa-thumb-tack"></i>
            <font color=7D26CD>置顶</font>
            <span class="post-meta-divider">|</span>
          {% endif %}
```
---
至此，博客置顶的方式就全部完成了
