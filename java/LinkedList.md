---
title: LinkedList底层分析
date: 2018-07-11 16:26:32
tags:
- list
categories:
- java
---
## 简介
***
>LinkedList与ArrayList一样是一个集合类，用于顺序存储元素允许元素为null。 在开发中ArrayList经常被用到， ArrayList 内部采用数组保存元素，适合用于随机访问比较多的场景，而随机插入、删除等操作因为要移动元素而比较慢。 LinkedList 内部采用双向链表的形式存储元素，随机访问比较慢，但是插入、删除元素比较快，LinkedList与ArrayList一样都是线程不安全的，所以要避免在多线程下使用，LinkedList实现了 Deque 所以它也可以作为一个双端队列。

[![LinkedList.png](https://i.loli.net/2018/07/11/5b45749ab5cd9.png)](https://i.loli.net/2018/07/11/5b45749ab5cd9.png)
## 属性
***
```JAVA
//链表节点数
transient int size = 0;
//指向第一个节点的指针
transient Node<E> first;
//指向最后一个节点的指针
transient Node<E> last;
```
属性较少就三个，根据上图应该很容易理解first,last的作用
<!--more-->
## 构造器
***
```JAVA
public LinkedList() {
}
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```
两个构造器，一个什么都没干，另一个调用addAll方法将集合里的所有元素插入链表中，后面会对addAll方法进行详细介绍
## 节点(Node)
***
```JAVA
private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
}
```
Node 是在 LinkedList 里定义的一个静态内部类，其内部结构包括一个数据item，一个后置指针next，一个前置指针prev
## 添加数据
***
### 头部添加
```JAVA
public void addFirst(E e) {
  linkFirst(e);
}
```
```JAVA
private void linkFirst(E e) {
        final Node<E> f = first;//将首节点赋值给f变量
        //创建节点将前置节点设置为null，后置节点指向f
        final Node<E> newNode = new Node<>(null, e, f);
        first = newNode;//将当前节点设置为首节点
        if (f == null)//代表list里还没有任何节点，所以首节点，尾节点都是当前创建的节点
            last = newNode;
        else
            f.prev = newNode;//将f的前置节点指向当前节点
        size++;//计数自增
        modCount++;//修改计数自增
  }
```
头部添加，首先是将原来的首节点赋值给一个临时变量，然后创建新的节点将其前置节点设置为null，值设置为要插入的数据，后置节点设置为f，然后将创建的节点设置首节点，如果当前集合里没有任何数据，则尾节点也将设置为当前创建的节点，否则将f的前置节点设置为当前创建的节点
### 尾部添加
```JAVA
public void addLast(E e) {
        linkLast(e);
  }
```
```JAVA
void linkLast(E e) {
        final Node<E> l = last;//将尾节点赋值给l变量
        //创建新节点，前置节点设置为l,后置节点设置为null
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;//将新创建的节点赋值给尾部节点
        if (l == null)//代表集合里还有任务数据
            first = newNode;//首节点也将是为当前节点
        else
            l.next = newNode;//将l后置节点设置为当前新创建的节点
        size++;//大小计数自增
        modCount++;//修改标识计数自增
  }
```
尾部添加首先是将尾部节点赋值给一个临时变量l，然后创建新的节点其前置节点为l,item为当前要插入的数据，后置节点设置为null，将尾部节点设置创建的节点，如果当前集合里还没有任何数据则首节点也要设置为当前节点，否则将l的后置节点设置为当前创建的节点
```JAVA
public boolean add(E e) {
        linkLast(e);
        return true;
  }
```
add方法默认就是调用的尾部添加方法，所以与`addLast`基本上是一样的
### 指定位置添加
```JAVA
public void add(int index, E element) {
        checkPositionIndex(index);//检查下标是否越界
        if (index == size)
            linkLast(element);//尾部插入
        else
            //指定位置前插入，node方法查找指定位置的节点，内部使用二分查找
            linkBefore(element, node(index));
  }
```
`node`方法通过二分查找法查找指定位置节点
```JAVA
Node<E> node(int index) {
        //size向右移一位，也就是size/2
        if (index < (size >> 1)) {//如果指定位置小于size/2则从首节点往后查找 
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {//从尾节点往前查找
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
  }
```
`linkBefore`方法在succ节点前，插入一个新节点e
```JAVA
void linkBefore(E e, Node<E> succ) {
        final Node<E> pred = succ.prev;//保存succ节点的前置节点
        //创建新节点，前置节点为pred，item为e,后置节点为succ
        final Node<E> newNode = new Node<>(pred, e, succ);
        succ.prev = newNode;//设置succ节点的前置节点为新创建的节点
        if (pred == null)//为空则succ为首节点
            first = newNode;//将新创建的节点设置首节点
        else
            pred.next = newNode;//将pred的后置节点设置新创建的节点
        size++;//大小计数自增
        modCount++;//修改标识计数自增
  }
```
指定位置添加数据，先检查下标是否越界，若越界则抛出异常，否则判断指定位置是否为尾部，若是则执行尾部添加数据方法，否则根据二分查找法查找到指定位置的节点，然后在它之前将创建的新节点插入
### addAll方法
```JAVA
public boolean addAll(Collection<? extends E> c) {
        return addAll(size, c);//以size为插入下标，插入集合c中所有元素
  }
```
```JAVA
public boolean addAll(int index, Collection<? extends E> c) {
        checkPositionIndex(index);//检查下标是否越界
        Object[] a = c.toArray();//集合转为数组
        int numNew = a.length;//新增元素数量
        if (numNew == 0)//新增元素数量为0则直接返回false
            return false;
        Node<E> pred, succ;
        if (index == size) {//指定位置为链表尾部
            succ = null;
            pred = last;//保存尾节点到pred
        } else {
            succ = node(index);//得到指定位置的节点
            pred = succ.prev;//保存指定位置节点的前置节点
        }
        for (Object o : a) {//循环遍历数组
            @SuppressWarnings("unchecked") E e = (E) o;
            Node<E> newNode = new Node<>(pred, e, null);//创建新节点
            if (pred == null)//代表指定位置为首节点
                first = newNode;//将新创建的节点设置为首节点
            else
                pred.next = newNode;//将指定位置节点的前置节点的后置节点指向新创建的节点
            pred = newNode;//将新创建的节点赋给pred为下次循环做准备
        }
        if (succ == null) {//代表是在尾部添加的数据
            last = pred;//将最后创建的节点设置尾节点
        } else {
            pred.next = succ;//将最后添加的节点的后置节点指向指定位置的节点
            succ.prev = pred;//将指定位置的节点的前置节点指向最后添加的数据节点
        }
        size += numNew;//更新大小
        modCount++;
        return true;
  }
```
addAll方法在指定下标插入集合里的所有元素，首先会判断下标是否越界，若未越界则将集合转为数组得到新添加元素的数量，如果新增元素数量为0则直接返回false，否则判断指定位置是否为链表尾部，若是则依次将集合里的元素添加到尾部，否则添加到指定位置节点的前面，最后更新size
## 删除数据
***
### 头部删除
```JAVA
public E removeFirst() {
        final Node<E> f = first;
        if (f == null)//判断首节点是否为空，若是则抛出异常
            throw new NoSuchElementException();
        return unlinkFirst(f);
  }
```
头部删除数据，首先判断链表首节点是否为null若是代表集合里还没有数据则抛出异常否则执行`unlinkFirst`方法，源代码中的remove()方法也是采用的头部删除
```JAVA
private E unlinkFirst(Node<E> f) {
        final E element = f.item;//保存首节点元素
        final Node<E> next = f.next;//保存首节点的后置节点
        f.item = null;//将元素置为null
        f.next = null; //后置节点置为null
        first = next;//将保存的后置节点提升为首节点
        if (next == null)//代表集合为空了
            last = null;
        else
            next.prev = null;//将后置节点的前置节点置为null
        size--;//大小计数自减
        modCount++;
        return element;
  }
```
`unlinkFirst`方法首先会保存链表首节点元素与首节点里的后置节点，然后将首节点的元素与后置节点置为null以便于垃圾回收，将保存的后置节点提升为首节点，如果保存的后置节点为null说明集合里没有数据，需要将链表尾节点置为null，若保存的后置节点不为null则将其前置节点置为null，最后对大小计数自减，修改标识自增并返回删除的元素
### 尾部删除
```JAVA
public E removeLast() {
        final Node<E> l = last;//保存尾部节点
        if (l == null)
            throw new NoSuchElementException();
        return unlinkLast(l);
  }
```
尾部删除，首先判断尾部节点是否为null，如果则抛出异常，否则执行`unlinkLast`方法
```JAVA
private E unlinkLast(Node<E> l) {
        final E element = l.item;//保存节点元素
        final Node<E> prev = l.prev;//保存尾节点的前置节点
        l.item = null;//将元素置空
        l.prev = null; // 将尾节点的前置节点置空
        last = prev;//设置保存的前置节点为链表的尾节点
        if (prev == null)//代表集合里没有数据
            first = null;//首节点了需要置为null
        else
            prev.next = null;//设置保存的前置节点的后置节点为null
        size--;//大小计数自减
        modCount++;
        return element;
  }
```
`unlinkLast`方法首先会保存链表尾节点元素与它的前置节点，然后将尾节点的元素与前置节点置为null以便于垃圾回收，将保存的前置节点设置为链表尾节点，如果保存的前置节点为null说明集合里没有数据，需要将链表首节点置为null，若保存的前置节点不为null则将其后置节点置为null，最后对大小计数自减，修改标识自增并返回删除的元素
### 删除指定元素
```JAVA
public boolean remove(Object o) {
        //要删除的是null节点(从remove和add 里 可以看出，允许元素为null)
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {//循环遍历
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
  }
```
移除指定元素，首先循环遍历链表找到要移除的数据，然后执行`unLink`方法
```JAVA
E unlink(Node<E> x) {
        final E element = x.item;//保存节点的元素
        final Node<E> next = x.next;//保存节点的后置节点
        final Node<E> prev = x.prev;//保存节点的前置节点

        if (prev == null) {//如果前置节点为null说明移除的是链表首节点
            first = next;//将后置节点提升为首节点
        } else {
            prev.next = next;//将保存的前置节点的后置节点指向保存的后置节点
            x.prev = null;//将移除节点的前置节点置为null利于垃圾回收
        }

        if (next == null) {//说明移除的是尾节点
            last = prev;//更新链表尾节点
        } else {
            next.prev = prev;//将保存的后置节点的前置节点指向保存的前置节点
            x.next = null;//将移除节点的后置节点置为null利于垃圾回收
        }

        x.item = null;//置空移除的元素
        size--;//大小计数自减
        modCount++;
        return element;
  }
```
`unlink`方法首先将要移除节点的元素item，前置节点，后置节点都保存到临时变量中，如果保存的前置节点等于null说明移除的是链表首节点，否则将保存的前置节点的后置节点指向保存的后置节点然后将移除节点的前置节点置为null利于垃圾回收，如果保存节点的后置节点为null说明移除的是尾节点，则更新链表的尾节点，否则将保存的后置节点的前置节点指向保存的前置节点，将移除节点的后置节点置为null利于垃圾回收
```JAVA
public boolean removeFirstOccurrence(Object o) {
        return remove(o);
 }
```
删除指定元素第一次出现的元素，其实就是执行了remove方法
```JAVA
public boolean removeLastOccurrence(Object o) {
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (x.item == null) {
                    unlink(x);
                    return true;
                }
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                if (o.equals(x.item)) {
                    unlink(x);
                    return true;
                }
            }
        }
        return false;
  }
```
删除指定元素最后一次出现的元素，内部是从链表的尾节点往前查找，若找到指定的元素则执行`unlink`方法
### 删除指定下标元素
```JAVA
public E remove(int index) {
        checkElementIndex(index);//判断下标是否越界，若越界则抛出异常
        return unlink(node(index));
    }
```
首先判断下标是否越界，若越界则抛出异常，否则根据`node`方法找到该下标的节点，然后执行`unlink`方法
## 修改元素
***
```JAVA
public E set(int index, E element) {
    checkElementIndex(index);//检查是否越界
    Node<E> x = node(index);//查找指定下标的元素
    E oldVal = x.item;//保存原元素值
    x.item = element;//将新值更新到节点上
    return oldVal;//返回原值
}
```
执行流程如下：
* 首先是检查是否越界，若越界则抛出异常
* 根据node方法查找指定下标的节点
* 保存节点的元素值，将元素节点值替换成新的
* 返回旧的元素值
## indexOf与lastIndexOf
***
```JAVA
public int indexOf(Object o) {
        int index = 0;
        if (o == null) {
            for (Node<E> x = first; x != null; x = x.next) {
                if (x.item == null)
                    return index;
                index++;
            }
        } else {
            for (Node<E> x = first; x != null; x = x.next) {
                if (o.equals(x.item))
                    return index;
                index++;
            }
        }
        return -1;
  }
```
从链表首节点往后开始查找指定元素，循环遍历链表若找到则返回下标，否则返回-1
```JAVA
public int lastIndexOf(Object o) {
        int index = size;
        if (o == null) {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (x.item == null)
                    return index;
            }
        } else {
            for (Node<E> x = last; x != null; x = x.prev) {
                index--;
                if (o.equals(x.item))
                    return index;
            }
        }
        return -1;
  }
```
从链表尾节点往前开始查找指定元素，循环遍历链表若找到则返回下标，否则返回-1
## 其它方法
***
```JAVA
public E peek() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
}

public E element() {
    return getFirst();
}
```
`peek`与`element`方法都是取出链表首节点元素
```
public E poll() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```
取出链表首节点元素并移除首节点
```
public boolean offer(E e) {
    return add(e);
}
```
添加元素到链表尾部
```
public boolean offerFirst(E e) {
    addFirst(e);
    return true;
}
```
添加元素到链表首部
```
public boolean offerLast(E e) {
    addLast(e);
    return true;
}
```
添加元素到链表尾部
```
public E peekFirst() {
    final Node<E> f = first;
    return (f == null) ? null : f.item;
 }
```
取出链表首节点元素
```
public E peekLast() {
    final Node<E> l = last;
    return (l == null) ? null : l.item;
}
```
取出链表尾节点元素
```
public E pollFirst() {
    final Node<E> f = first;
    return (f == null) ? null : unlinkFirst(f);
}
```
取出链表首元素并移除首元素
```
public E pollLast() {
    final Node<E> l = last;
    return (l == null) ? null : unlinkLast(l);
}
```
取出链表尾元素并移除尾元素
```
public void push(E e) {
    addFirst(e);
}
```
添加元素到链表首部
```
public E pop() {
    return removeFirst();
}
```
移除链表首元素并返回
## 总结
* LinkedList 的底层结构是一个带头/尾指针的双向链表，可以快速的对头/尾节点进行操作
* 对比ArrayList而言，在指定位置插入和删除元素的效率较高，但查找效率就没有ArrayList高
* LinkedList通过下标查找节点时会根据二分查找法进行搜索
* LinkedList批量添加数据时是循环遍历数组元素插入节点，而ArrayList则是靠拷贝数组来完成
* LinkedList与ArrayList一样都是线程不安全的