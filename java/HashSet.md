---
title: HashSet原理分析
tags:
  - set
categories:
  - java
abbrlink: e40c8608
date: 2018-07-15 14:59:20
---
## 概述
HashSet实现了Set接口，由哈希表（实际上是一个HashMap实例）支持。它不保证set 的迭代顺序；它并不保证随着时间的推移，秩序将保持不变。此类允许使用null元素,是一个不允许存储重复元素的集合
## 成员变量
```JAVA
 private transient HashMap<E,Object> map;
 private static final Object PRESENT = new Object();
```
* map:用于存放数据
* PRESENT: 所有存入的map数据的value值
<!--more-->
## 构造器
```JAVA
   public HashSet() {
        map = new HashMap<>();
    }

    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
```
通过构造器可以发现使用的是HashMap来初始化map，但其中有一个包访问级别的构造器，使用的是`LinkedHashMap`，其主要是对LinkedHashSet的支持。
## add方法
```JAVA
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }
```
通过add方法可以发现，存入的数据其实是作为了map的键，而真正的值是`PRESENT`,所以这就保证了存入的数据是没有重复的
## 总结
* 由于HashSet是借助于HashMap来实现的，所以HashMap会出现的问题,HashSet一样会出现
* 在用HashSet保存对象的时候，一定要正确的重写其equals和hashCode方法，以保证放入的对象的唯一性