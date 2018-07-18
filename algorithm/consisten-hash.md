---
title: 一致性Hash分析
date: 2018-07-18
tags:
- hash
- algorithm
categories:
- java
---
## 场景描述
在开发中经常会用到缓存来减轻数据库的压力，比较常见的就是使用redis，但当数据达到一定量时一台缓存服务器已经无法满足现在的需求，所以需要进行分布存储，假设现在有3台缓存服务器，那么该如何将数据均匀的分布到这3台服务器上了。
## 取模运算方式
将传入的Key按照`index = hash(key) % len`方式计算出需要存放的服务器节点，其中len是服务器数量，通过取模运算就能将数据均匀的分布到不同的服务器上。但这种方式有缺陷，如果某一天老板说再加一台服务器，此时服务器数就变成了4，那么通过上面方式来获取数据很有可能不会命中，因为重新计算了key，换句话说当服务器数量发生改变时，所有缓存在一定时间内是失效的，当应用无法从缓存中获取数据时则会向后端服务器请求数据。同理，假设缓存服务器中突然有一台务器出现了故障，无法进行缓存，那么我们则需要将故障机器移除，但是如果移除了一台缓存服务器，那么缓存服务器数量从4台变为3台，如果想要访问数据，这条数据的缓存位置必定会发生改变，之前缓存的数据也失去了缓存的作用与意义，由于大量缓存在同一时间失效，造成了缓存的雪崩，此时后端服务器将会承受巨大的压力，整个系统很有可能被压垮。
<!--more-->
## 一致性Hash算法
一致 Hash 算法是将所有的哈希值构成了一个环，其范围在 `0 ~ 2^32-1`，就像吃的饼子一样，饼子我们可以理解为是由0~60个点组成的圆，那么在这里我们可以理解为由`0~2^32-1`个点组成的圆，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-5d57d235a45ff089.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后通过hash计算将各个节点分配到这个环上，hash计算时可以通过服务器IP+服务器名称等作为唯一key来进行hash(key)计算，假如分配后的图如下：![](https://upload-images.jianshu.io/upload_images/13023122-901a27aa013d535d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
假如现在我们需要进行数据缓存，那么我们使用hash(key)方式将key映射到环上，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-7f5413e714c3c0da.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在已经把服务器与数据都映射到这个环上了，但是数据k1会存入哪一台服务器了，这里的数据k1将会存储在s2服务器上，因为从数据在环上的的位置开始顺时针方向找到的第一个服务器就是s2,所以上图中的数据k1会缓存到s2服务器上，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-89e79acab91e7a88.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 容错性
假如现在我们有一台服务器s1宕机了，那么此时受影响的其实只有k3数据它将会重新映射到S2服务器上，k1,k2其实是没有发生变化的，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-b296891ea3c18d8f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 扩展性
假如现在我们需要在之前3台服务器的基础上再加一台服务器s4，恰好映射在s2与s3之间，那么此时受影响的只有k2，而k1与k3没有发生变化，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-49aa1433cecc9df5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 环的偏斜
在上面的图中我们都是以最好的情况来分析的，想像的是服务器是均匀的分布到这个环上面，然后实际开发中，其实并不一定是均匀分布的，会出现映射的节点偏向某一个方向的情况，这种情况很容易造成数据分布不均匀，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-e15db68df38ab7f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)如上图所示，数据分布在S1服务器上的最多
## 虚拟节点
为了应对环的偏斜我们可以添加虚拟节点来弥补，虚拟节点其实就是实际服务器节点的复制品，一个实际节点可以对应多个虚拟节点，我们在生成虚拟节点时，可以在实际服务器IP地址后面加上序号号通过hash计算来生成，如下图所示：![](https://upload-images.jianshu.io/upload_images/13023122-6080e76a2fdc81d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图中s11,s21,s31就是生成的虚拟节点，这样缓存的分布就均匀得多了，还可以添加更多的虚拟节点来解决环的偏斜，虚拟节点越多缓存分布越均匀。
## 总结
* 在进行分布式存储开发中，最好使用一致性Hash来均匀的存放数据
* 当服务器较少时，可以添加虚拟节点来解决环的偏斜，从而使得数据均匀的分布到服务器上。