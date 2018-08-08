---
title: JAVA垃圾回收机制
categories:
  - java
abbrlink: 40a57016
date: 2018-07-26 00:00:00
---
围绕三个点进行说明：
* 哪些对象需要回收
* 什么时候回收
* 怎么回收
## 哪些对象需要回收
要想知道哪些对象需要回收，就得知道哪些对象还是处于存活状态，java中有两算法来判断对象是否还处于存活状态
### 引用计数算法
在堆中每个对象实例都有一个引用计数，当一个对象被创建时，就将该对象实例分配给一个变量，该变量计数设置为1，当这个对象被赋值给其它变量时则计数加1，当那个变量超过了生命周期或者被设置为一个新值时则计数减1，当计数为0时则认为该对象死亡可以回收了。

虽然这种算法简单效率高，但是在循环引用的时候会出现无法回收的问题。
### 可达性分析算法
从一个节点GC ROOT开始，录找对应的引用节点，找到后继续去寻找它的引用节点，直到寻找完毕，可以把这一寻找过程理解为一个串。当一个对象通过这个串无法到达GC Root时则可以确认该对象已死亡可以被回收。
![](https://upload-images.jianshu.io/upload_images/13023122-a872b4a45336e57f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过上图可以看出object 5,6,7是无法到达GC Root的所以他们是可以被回收的对象
<!--more-->
可作为GC Roots的对象包括下面几种：
>* 虚拟机栈中所引用的对象;
>* 方法区中类静态属性引用的对象；
>* 本地方法栈中JNI（Native方法）引用的对象。
## 什么时候回收
一个对象实例化时会先去看Eden有没有足够的空间，如果有则不进行垃圾回收 ,对象直接在Eden中，若Eden内存已满,会进行一次垃圾回收清除非存活对象，然后将Eden中存活的对象放入Survivor区，最后整理Survivor的两个区。

如果Eden与Survivor的两个区都满了，则查看老年代区是否已满，若没满则将部分存活区（Survivor）的活跃对象存入老年代，然后将Eden的活跃对象放入存活区（Survivor）。

如果老年代内存不足则进行一次full gc，之后老年代会再进行判断 内存是否足够,如果足够同上，否则会抛出OutOfMemoryError。
## 怎么回收
### 标记-清除算法
从GC Root开始扫描，对存活的对象进行标记，标记完毕后，再扫描整个空间中未被标记的对象，然后进行回收。![回收前](https://upload-images.jianshu.io/upload_images/13023122-bd10bd43dc8a958f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![回收后](https://upload-images.jianshu.io/upload_images/13023122-0abd9c1d6dcc947d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)从上面的图中可以看出，回收后内存空间不是连续的，这就导致产生了很多碎片，如果下一次申请的空间较大时无法找到足够的连续内存所以就得提前触发一次垃圾收集。
### 复制算法
复制算法是将可用的内存分为两块区域，每次只用其中的一块，当发生垃圾回收时，会将存活的对象复制到另一块未使用的区域，然后对之前的区域进行回收。![复制算法](https://upload-images.jianshu.io/upload_images/13023122-5d81e849eb1fcc29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 标记-整理算法
复制算法在对象存活率较高的场景下要进行大量的复制操作，效率很低。另一个是老年代存放的都是不容易被回收的对象，再加上内存空间的限制所以一般是不会采用复制算法。老年代采用的与标记-清除算法类似的标记整理算法。

标记整理算法在对对象进行标记时与标记清除算法一样，区别在于它是让所有存活对象都移动到一端，然后直接清理掉边界以外的内存。![回收前](https://upload-images.jianshu.io/upload_images/13023122-bd10bd43dc8a958f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)![回收后](https://upload-images.jianshu.io/upload_images/13023122-2bcacedb83deaa6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 分代回收算法
* 新生代：分为三个区，Eden和两个Survivor区，整个流程是，Eden满了会向将Eden中的存活对象放入Survivor0中，Survivor0满了则将Eden与Survivor0中存活的对象放入Survivor1中，清空eden和这个survivor0区，将Survivor0与Survivor1进行交换，重复这样的流程。当survivor1区不足以存放 eden和survivor0的存活对象时，就将存活对象直接存放到老年代。若是老年代也满了就会触发一次Full GC。
* 老年代：老年代中存放的都是一些生命周期较长的对象。当老年代满时会触发Full GC
在jdk8之前还有持久代，从jdk8开始已经将持久代（PermGen Space） 干掉了，取而代之的元空间（Metaspace）。Metaspace占用的是本地内存，不再占用虚拟机内存。
在新生代中采用的是复制算法，简单高效。在老年代中则是采用标识-清除或者是标记-整理算法。
