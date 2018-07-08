---
title: 关于IDEA再从git或者svn上导入项目时不能加载字模块的问题
date: 2018-07-08 23:26:49
tags: [IDEA,开发日记,git]
---

### 关于IDEA再从git或者svn上导入项目时不能加载字模块的问题

> 最近入职新公司，很多东西也都算是要从头学起。在之前公司用的都是eclipse，这边要求用IDEA，其实很早就知道这是一个非常强大的编译器，但平时没有机会使用，现在有机会用这个还是挺开心的。

由于公司使用gitlab，在注册好账号导入代码的时候遇到一个情况，就是直接用IDEA的git工具导入的话会出现，maven项目的子模块无法被识别以及被管理的情况。事实上eclipse也有同样的问题。现在只说下使用IDEA遇到这个情况的解决办法。

<!--more-->

#### 有两种解决方式。
##### 1、手动将module添加到项目管理：
- 打开文件选项中的项目结构（快捷键ctrl+alt+shift+s）
[![oLFwe.png](https://s1.ax2x.com/2018/07/08/oLFwe.png)](https://simimg.com/i/oLFwe)
- 选择 模块-加号-导入module，手动将自己需要的模块一一导入进去
[![oLTGd.png](https://s1.ax2x.com/2018/07/08/oLTGd.png)](https://simimg.com/i/oLTGd)
##### 2、先将项目通过命令行导入到本机，然后通过IDEA的New Project from Existing Sources导入本地项目进来，这个直接就能够对所有模块进行代码管理了
[![oLXER.md.png](https://s1.ax2x.com/2018/07/08/oLXER.md.png)](https://simimg.com/i/oLXER)
[![oLm7r.md.png](https://s1.ax2x.com/2018/07/08/oLm7r.md.png)](https://simimg.com/i/oLm7r)

> 从eclipse转到IDEA前几天是最艰难的，因为很多习惯不是说改就能改掉的，工具的使用总得需要一个学习的时间，但是等这段时间过去，后面一定会体会到IDEA的强大。

