---
title: HashMap分析
date: 2018-07-09 18:22:41
tags: 
- HashMap
categories:
- java
---
## 1.基本组成成员
***
```JAVA
//默认容量,数组长度16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
//最大容量,数组最大长度
static final int MAXIMUM_CAPACITY = 1 << 30;
//负载因子，默认0.75f
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//将链表转为红黑树的阀值，默认为8
static final int TREEIFY_THRESHOLD = 8;
//将红黑树转为链表的阀值，默认为6
static final int UNTREEIFY_THRESHOLD = 6;
//Node类型的数组
transient Node<K,V>[] table;
//实际存储的键值对个数
transient int size;
//用于迭代防止结构性破坏的标量
transient int modCount;
//扩容阀值，threshold = loadFactor * capacity
int threshold;
//负载因子变量
final float loadFactor;
```
![](https://i.loli.net/2018/07/09/5b43131ce94bf.png)
>HashMap 是 Map 的一个实现类，它代表的是一种键值对的数据存储形式,Key 不允许重复出现,但可以储存null键值,在jdk8中其内部是由数组+链表+红黑树来实现
## 2.构造方法
***
```java
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
```
HashMap一共有4个构造方法，如果用户没有传入initialCapacity 和loadFactor这两个参数，会使用默认值，`initialCapacity`默认为16，`loadFactory`默认为0.75，初始的时候并没有为`Node`数组分配内存空间，而是在put的时候进行分配
<!--more-->
## 3.hash的计算
***
```JAVA
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
 }
```
这个函数大概的作用就是：高16bit不变，低16bit和高16bit做了一个异或，而Node数组下标计算则是按照下面方式进行计算：
```JAVA
(n - 1) & hash
```
## 4.put方法具体实现
***
```JAVA
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //判断是否初始了，若没有则初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //判断当前下标是否有数据，没有则存入
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        //有数据则放入链表尾部
        else {
            Node<K,V> e; K k;
            //当前节点与要插入的节点为同一个，此时仅仅是修改操作
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //当前节点是红黑树节点，则以红黑树方式插入
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //遍历链表，将节点插入尾部
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        //判断是否需要将链表变为红黑白树，大于等于8时进行转换
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        //添加后，是否达到扩容阀值，如果是则需要进行扩容操作
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
}
```
执行流程如下：
* 判断table数级是否初始化了，如果没有则进行初始化`resize()`
* 根据`(n - 1) & hash`得到数组下标，判断当前索引是否存储得有数据，如果没有则创建node节点并插入
* 如果上面步骤有数据则判断当前节点是否与将要插入的节点为同一个，如果是则仅仅是修改操作，否则执行下面步骤
* 判断当前节点是否为红黑树节点，如果是则按照红黑树节点方式插入，否则执行下面步骤
* 遍历链表，将要插入的节点放入尾部，需要判断是否需要转换为红黑树，若满足要求则进行转换
* 最后添加节点后需要判断是否需要进行扩容操作
## 5.get具体实现
***
```JAVA
public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
}
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //对table进行校验，判断是否为空，长度是否大于0，取出的数据是否不等于null
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //判断当前下标里的第一个节点是否就是所要查找的，若是则直接返回
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            //第一个节点不是所查找的则遍历查找
            if ((e = first.next) != null) {
                // 若节点是红黑树节点，则按红黑树方式进行查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                //遍历链表
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
}
```
执行流程如下：
* 首选对table数组进行校验，判断是否不等于null,长度是否大于0，取出的数据是否不等于null,若条件满足则执行下面步骤
* 判断上一步取出的节点里的hash与key是否与当前查找的hash，key相等，如果相等则直接返回当前节点，否则继续往下执行
* 根据第一节点判断是否为红黑树节点，若是则按红黑树方式进行查找，否则遍历链表进行查找，直到找到为止
## 6.remove具体实现
***
```JAVA
public V remove(Object key) {
        Node<K,V> e;
        return (e = removeNode(hash(key), key, null, false, true)) == null ?
            null : e.value;
}
final Node<K,V> removeNode(int hash, Object key, Object value,
                               boolean matchValue, boolean movable) {
        Node<K,V>[] tab; Node<K,V> p; int n, index;
        //对table进行校验，判断是否为空，长度是否大于0，取出的数据是否不等于null
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (p = tab[index = (n - 1) & hash]) != null) {
            Node<K,V> node = null, e; K k; V v;
            //判断当前下标里的第一个节点是否就是所要查找的，若是则赋值给局部变量node
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                node = p;
            else if ((e = p.next) != null) {//若不是所要找的则判断下一节点是否为不等于null
                if (p instanceof TreeNode)//如果第一个节点为红黑树节点则按红黑树方式查找
                    node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
                else {
                    //遍历链表查找
                    do {
                        if (e.hash == hash &&
                            ((k = e.key) == key ||
                             (key != null && key.equals(k)))) {
                            node = e;
                            break;
                        }
                        p = e;
                    } while ((e = e.next) != null);
                }
            }
            //得到所要移除的节点与传入的value进行比较判断是否为同一个
            if (node != null && (!matchValue || (v = node.value) == value ||
                                 (value != null && value.equals(v)))) {
                if (node instanceof TreeNode)//按照红黑树方式移除
                    ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
                else if (node == p)//所要移除的就是首节点则直接将下一个节点提为首节点
                    tab[index] = node.next;
                else
                    p.next = node.next;//将所要移除节点的后置节点挂到它前置节点的next上
                ++modCount;
                --size;
                afterNodeRemoval(node);
                return node;
            }
        }
        return null;
}
```
执行流程如下：
* 前面部分与get方法类似，先去查找所要称除的节点，唯一不同的是get方法是查找到后直接返回，这里是将值赋值给了局部变量node
* 得到所要移除的节点后，将其与传入的参数value进行比较，如果条件满足则继续往下面执行
* 判断所要移除的节点是否为红黑树节点，若是则按红黑树方式进行移除
* 上一步若不满足，则判断所要移除的节点是否为首节点，若是则将其后置节点提升为首节点
* 上一步若不满足，则将所要移除节点的后置节点挂到它前置节点的next上
## 7.resize方法具体实现
***
```JAVA
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            // 超过最大值就不再扩充了
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            //未超过最大值则扩容为原来的2倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            //使用了initialCapacity参数的构造器，首次扩容则会到这里
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            //使用了未带参数的构造器，首次扩容则会到这里
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        //计算新的resize上限
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //旧数组不为 null，这次的 resize 是一次扩容行为
        if (oldTab != null) {
            //遍历方式把每个bucket都移动到新的buckets中
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)//首节点则直接分配
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)//红黑树节点方式分配
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {//索引值不会发生变化
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {//索引值发生变化，原索引+oldCap
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        //下面主要是将链表上的数据按照e.hash & oldCap不等0的将分配到原索引+oldCap的链表上  
                        //原索引存入
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        //原索引+oldCap存入
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
}
```
执行流程如下：
* 如果是第一次扩容，首先会根据选用的构造器来进入不同的条件，上面代码中已有注释
* 循环遍历，将原索引链表上的一部数据分配到新索引链表上，计算方式为（e.hash & oldCap） == 0，如果为0则不发生改变，否则将分配到新索引链表上去，这样做的好处是避免了重新计算hash值，将数据均匀分散减少碰撞几率
##总结：
>* 根据分析可知,HashMap内部的储存依赖hash值的计算，所以选用String，Integer这些类做为键会提高HashMap的效率,因为他们本身具备作为键值的天生条件( hashCode(),equal()方法)
>* HashMap是线程不安全的，不要在并发的环境中同时操作HashMap，建议使用ConcurrentHashMap
>* 由于扩容是一个特别耗性能的操作，所以在使用HashMap的时候最好估算下map大小，给一个值避免频繁的进行扩容