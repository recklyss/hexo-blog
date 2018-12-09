---
title: Java中map新的骚操作
date: 2018-12-09 23:03:05
tags: [开发日记,Java拾遗]
---
#### 在Java8中对于Map的操作新增的compute之类的方法，对于开发中很有帮助，在此处整理一下其用法，以及方法之间的异同，具体的先总结一下如下：

> #### 总结
> `computeIfPresent` 就是根据方法来，返回方法中的值对原值进行替换，新的为null就删除键值对，但是原值为null新值不为null依然返回null
> `computeIfAbsent`  就是根据旧值来，旧的没有再根据方法返回的来，旧值存在就返回旧值
> `compute`          就是两者结合，新值为null，就删除键值对；新值不为null就进行替换。


<!--more-->
```
package test;

import java.util.HashMap;

public class MapTest {

    public static void main(String[] args) {
        HashMap<Integer, String> map = new HashMap<>();
        map.put(1,"zhang");
        // computeIfPresent 根据之前的key/value 如果oldValue 不为null 则根据提供的方法返回一个新的值 并进行新值对旧值的替换
        System.out.println(" 1 ---> " + map.computeIfPresent(1,(key,value)->{
            return key + value;//原值不为null新值不为null 新值替换旧值
        }));
        // 否则删除键值对
        System.out.println(" 2 ---> " + map.computeIfPresent(1,(key,value)->{
            return null;//原值不为null新值为null 删除键值对
        }));
        map.put(1,null);
        System.out.println(" 3 ---> " + map.computeIfPresent(1,(key,value)->{
            return "jiaheng";//原值为null 不做更改
        }));
        // computeIfAbsent 根据之前的key 如果旧值为空或者key不存在 就按照方法用新值替换旧值 新值为null不做替换
        map.put(1,"zhang");
        System.out.println(" 4 ---> " + map.computeIfAbsent(1,k->{
            return null;// 不会被替换旧值 返回原值
        }));
        System.out.println(" 5 ---> " + map.computeIfAbsent(2,k->{
            k = k*k;
            return k.toString();// key=2不存在 直接新建并存入新值
        }));
        // compute类似于computeIfAbsent和computeIfPresent的合体
        map.put(1,null);
        System.out.println(" 6 ---> " + map.compute(1,(k,v)->{
            return "张";// 原值为null新值不为null 新值替换旧值 此处与computeIfPresent不同
        }));
        System.out.println(" 7 ---> " + map.compute(1,(k,v)->{
            v = (k*10) + v;
            return v;// 新值不为null 替换旧值
        }));
        System.out.println(" 8 ---> " + map.compute(1,(k,v)->{
            return null;// 新值为null 删除键值对
        }));

        // 总结
        // computeIfPresent 就是根据方法来，返回方法中的值对原值进行替换，新的为null就删除键值对，但是原值为null新值不为null依然返回null
        // computeIfAbsent  就是根据旧值来，旧的没有再根据方法返回的来，旧值存在就返回旧值
        // compute          就是两者结合，新值为null，就删除键值对；新值不为null就进行替换。
    }

}

```
