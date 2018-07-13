---
title: LinkedHashMap底层分析
date: 2018-07-12 17:37:34
tags:
- map
categories:
- java
---
## 简介
***
由于LinkedHashMap是在HashMap之上进行的，所以要理解LinkedHashMap需要先了解HashMap的原理。对HashMap的理解可以参考之前写的[HashMap分析](http://cqjokers.top/2018/07/09/java/HashMap/)，LinkedHashMap继承了HashMap，所以LinkedHashMap其实也是散列表的结构，不同的是LinkedHashMap是运用双向链表而HashMap是运用单向链表，通过源代码中的注释可以知道它使用链表来维护数据的有序性。
[![LinkedHashMap.png](https://i.loli.net/2018/07/12/5b46c00fabed9.png)](https://i.loli.net/2018/07/12/5b46c00fabed9.png)
## 属性
***
```JAVA
//内部静态类，元素的存储结构，继承至HashMap.Node,扩展为一个双向链表
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
    }
}
//头指针
transient LinkedHashMap.Entry<K,V> head;
//尾指针
transient LinkedHashMap.Entry<K,V> tail;
//此属性可以控制元素顺序，默认false,顺序为插入时的顺序
//为true时对集合里的数操作时会将数据放入尾部，通过它可以实现LRU缓存
final boolean accessOrder;
```
由于LinkedHashMap继承了HashMap所以其内部属性就比较少，通过head,tail来保存首尾元素，静态类Entry继承Node,并扩展为一个双向链表，通过`accessOrder`我们可以控制元素的顺序
<!--more-->
## 元素创建
***
```JAVA
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);//创建新节点
    linkNodeLast(p);//链接节点
    return p;
}

//红黑树方式创建节点
TreeNode<K,V> newTreeNode(int hash, K key, V value, Node<K,V> next) {
    TreeNode<K,V> p = new TreeNode<K,V>(hash, key, value, next);
    linkNodeLast(p);
    return p;
}
```
我们知道在put新元素时会通过newNode创建新节点，与HashMap不同的是这里创建节点后会将节点与前一个关联起来
```java
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;//保存尾部节点
    tail = p;//修改尾部节点为当前新节点
    if (last == null)//如果之前的尾部节点为空说明集合里还没有元素
        head = p;//修改头节点为当前新节点
    else {
        p.before = last;//设置当前节点的前置节点为原尾部节点
        last.after = p;//设置原尾部节点的后置节点为当前节点
    }
}
```
通过`linkNodeLast`方法可以将新元素关联到链表尾部，并将其与前一个节点关联上
## 关键方法
***
在HashMap中预留了三个方法给LinkedHashMap,通过这三个方法可以保证链表的插入、删除的有序性
```JAVA
void afterNodeAccess(Node<K,V> p) { }//访问的回调
void afterNodeInsertion(boolean evict) { }//插入后的回调
void afterNodeRemoval(Node<K,V> p) { }//删除后的回调
```
### afterNodeAccess方法
```JAVA
void afterNodeAccess(Node<K,V> e) { // 移动节点到尾部
        LinkedHashMap.Entry<K,V> last;
        //如果accessOrder为true且当前访问的节点不是尾节点
        if (accessOrder && (last = tail) != e) {
            //节点e强转成双向链表节点p,保存前置节点与后置节点
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;//将后置节点置为null
            if (b == null)//说明原节点是首节点
                head = a;//将原节点的后置节点提升为首节点
            else
                b.after = a;//更新p的前置节点b的后置节点为a
            if (a != null)//p的后置节点不为null
                a.before = b;//更新p的后置节点a的前置节点为b
            else//说明原节点就是尾节点
                last = b;//更新尾节点为原节点的前置节点b
            if (last == null)//如果尾节点为空说明链表中就只有一个节点就是p
                head = p;//更新头节点为p
            else {
                p.before = last;//更新当前节点p的前置节点为原尾节点last
                last.after = p;//更新last的后置节点是p
            }
            tail = p;//更新p为链表尾节点
            ++modCount;
        }
    }
```
通过afterNodeAccess方法将当前节点移至链表尾部，但有一个条件就是当accessOrder为true时才会执行这个操作，在HashMap的putVal方法中，就调用了这个方法。
### afterNodeInsertion方法
```JAVA
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }

    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```
该方法会在链表中插入一个新节点的时候调用，目的是删除链表的首节点，当然需要重写`removeEldestEntry`方法因为默认是返回false，通过`afterNodeAccess`方法与`afterNodeInsertion`方法我们可以实现一个LRU缓存
### afterNodeRemoval方法
```JAVA
    void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
```
`afterNodeRemoval`方法主要目的是在移除数据后将双向链表中该节点一并移除掉
### get方法
```JAVA
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)
            afterNodeAccess(e);
        return e.value;
    }
```
该方法与HashMap里的get方法不同的是当`accessOrder`为true时会回调`afterNodeAccess`方法
### getOrDefault方法
```JAVA
    public V getOrDefault(Object key, V defaultValue) {
       Node<K,V> e;
       if ((e = getNode(hash(key), key)) == null)
           return defaultValue;
       if (accessOrder)
           afterNodeAccess(e);
       return e.value;
   }
```
该方法会根据key去查找数据，如果没有找到则返回默认值`defaultValue`，若找到则返回节点数据，最后判断`accessOrder`是否为true，若是则执行afterNodeAccess方法
### containsValue方法
```JAVA
    public boolean containsValue(Object value) {
        for (LinkedHashMap.Entry<K,V> e = head; e != null; e = e.after) {
            V v = e.value;
            if (v == value || (value != null && value.equals(v)))
                return true;
        }
        return false;
    }
```
该方法比HashMap里的效率要高，HashMap里使用了两个循环，这里只对链表从前往后查找
## 总结
>* LinkedHashMap继承了HashMap，所以HashMap存在的问题它也一样会存在
>* 可以利用LinkedHashMap构造一个LRUCache，条件是需要将accessOrder设置为true，并重写`removeEldestEntry`方法
>* LinkedHashMap插入的数据是有序的，而HashMap存入的数据是无序的