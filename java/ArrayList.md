---
title: ArrayList底层分析
date: 2018-07-10 17:08:41
tags:
- list
categories:
- java
---
## ArrayList简介
***
* ArrayList底层是基于数组实现，内部提供了一个Object数组且该数组是不可序列化的，可以动态的增加和减少元素
* ArrayList实现了Serializable接口，因此它支持序列化，能够通过序列化传输，实现了RandomAccess接口，支持快速随机访问，实际上就是通过下标序号进行快速访问，实现了Cloneable接口，能被克隆，实列了List接口提供了相关的添加、删除、修改、遍历等功
* ArrayList是线层不安全的，所以应该避免在多线程上使用
## ArrayList属性
***
```JAVA
  //默认容量10
private static final int DEFAULT_CAPACITY = 10;
private static final Object[] EMPTY_ELEMENTDATA = {};
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
//存入元素的数组，不可序列化 
transient Object[] elementData;
//实际数据数量
private int size;
```
>虽然ArrayList实现了Serializable接口，但`elementData`并没有参与序列化中，为了达到反序列时`elementData`有值，ArrayList源码中实现了自己的readObject和writeObject方法，序列化时直接将size和element写入ObjectOutputStream；反序列化时调用readObject，从ObjectInputStream获取size和element，再恢复到elementData，之所以不让`elementData`参与序列化是因为底层数组并不能保证每一个位置都存储得有元素，所以为了避免造成空间浪费，采用上诉的方式来实现序列化时，就可以保证只序列化实际存储的那些元素，而不是整个数组
<!--more-->
## ArrayList构造函数
***
```JAVA
//构造一个指定初始容量的空列表
public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {//大于0则创建指定容量的空数组
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {//等于0，将空数组赋给elementData
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
}
//无参构造函数,默认容量为10
public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
//构造一个包含指定 collection 的元素的列表
public ArrayList(Collection<? extends E> c) {
        //将collection对象转换成数组，然后赋给elementData
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
           //c.toArray很有可能返回的不是Object[].class，可以查看BUG编号为6260652
            if (elementData.getClass() != Object[].class)//判断类型是否相等，不相等则进行下面操作
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // 直接将空对象EMPTY_ELEMENTDATA的地址赋给elementData
            this.elementData = EMPTY_ELEMENTDATA;
        }
}
```
## add操作
***
```JAVA
public boolean add(E e) {
        ensureCapacityInternal(size + 1); 
        elementData[size++] = e;
        return true;
}
//第一次添加元素时，elementData 长度会设置为10
private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
}
private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;//修改标识自增1
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
}
//扩容
private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
}
```
执行流程如下：
* `ensureCapacityInternal`方法保证当前数组大小+1后能存入下一个数据
* 如果当前数组长度+1大于原数组长度，则执行扩容操作，将容量扩为原来的1.5倍(newCapacity = oldCapacity + (oldCapacity >> 1)),并将修改标识`modCount`自增1
* 将新元素添加到位于size的位置上后返回TRUE
```JAVA
public void add(int index, E element) {
        //确保index小于当前size且大于0，不满足则抛出异常
        rangeCheckForAdd(index);
        //此处与上面add(E e)方法调用一致
        ensureCapacityInternal(size + 1);  // Increments modCount!!
         //将index位置以及后面的元素往后移一个位置
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        //将新元素插入指定位置
        elementData[index] = element;
        size++;
}

private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```
这个方法其实add(E e)类似，该方法可以指定位置插入元素，具体的执行逻辑如下：
* 先执行`rangeCheckForAdd`方法以确保要插入的索引下标小于数组长度且大于0，如果不满足则抛出异常
* ensureCapacityInternal(size + 1);此步骤与add(E e)方法一致
* 将现有数组要插入的位置上的元素以及后面的元素都往后面移一个位置，完在后将新元素插入指定位置，将数组长度自增1
## get方法
***
```JAVA
 public E get(int index) {
        //检查下标是否大于数组长度，若是则抛出数组越界异常
        rangeCheck(index);
        return elementData(index);
  }

private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
  }
```
返回指定位置上的元素
## set方法
***
```JAVA
public E set(int index, E element) {
        //检查下标是否大于数组长度，若是则抛出数组越界异常
        rangeCheck(index);
        E oldValue = elementData(index);//得到原元素
        elementData[index] = element;//修改旧元素为新元素
        return oldValue;//返回旧元素
}
```
set方法将指定位置元素替换成新元素并返回原来的元素
## addAll方法
***
```JAVA
public boolean addAll(Collection<? extends E> c) {
        Object[] a = c.toArray();//转换为数组
        int numNew = a.length;//得到数组长度
        //确保当前size+numNew还有足够的容量存入数据
        ensureCapacityInternal(size + numNew);  // Increments modCount
        //插入数据
        System.arraycopy(a, 0, elementData, size, numNew);
        size += numNew;
        return numNew != 0;
  }
```
执行流程如下：
* 首先将集合转为数组
* 通过`ensureCapacityInternal`方法来确保当前size+numNew还有足够的容量来存入数据
* 将数组里的数据拷贝到`elementData`数组中
```JAVA
public boolean addAll(int index, Collection<? extends E> c) {
        rangeCheckForAdd(index);//判断指定位置是否越界
        Object[] a = c.toArray();//转换为数组
        int numNew = a.length;
        //确保当前size+numNew还有足够的容量存入数据
        ensureCapacityInternal(size + numNew);  // Increments modCount
        int numMoved = size - index;
        if (numMoved > 0)
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);//先将指定位置以及后面的数据移到 index + numNew位置
        //将新的元素拷贝到指定位置
        System.arraycopy(a, 0, elementData, index, numNew);
        size += numNew;
        return numNew != 0;
}
```
执行流程如下：
* 判断给的位置是否越界，若越界则抛出异常
* 将集合转换为数组，并进行扩容
* 如果指定的位置不是`elementData`数组最后一个则进行位置移动操作
* 最后将新的元素拷贝到`elementData`数组中
## remove方法
***
```JAVA
public E remove(int index) {
        rangeCheck(index);//索引检查，判断是否越界
        modCount++;//修改标识自增
        E oldValue = elementData(index);//获得元素
        int numMoved = size - index - 1;
        if (numMoved > 0)//如果不是最后一个元素则进行拷贝操作
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
        return oldValue;
}
```
执行流程如下：
* 首先检查指定下标是否超过数组大小，若超过则抛出数组越界异常
* 将modCount自增，获得指定位置的元素，方便最后返回
* 如果要移除的不是最后一个元素则需要进行数组拷贝操作，将指定位置后面的元素向前移一位
* 最后将最后一个索引位置设置为NULL好让垃圾回收器回收，并返回移除的元素
```JAVA
public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
}
```
循环遍历数组里所有对象，得到所要移除对象所在索引位置，然后调用fastRemove方法，执行remove操作
```JAVA
private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
  }
```
将所要移除的元素后面的所有元素往前移一位，并将最后一个位置的元素置空
## indexOf方法
***
```JAVA
public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
  }
```
循环遍历`elementData`数组中的每一个元素进行比较，若找到则返回元素所在下标，否则返回-1，源码中的`contains`方法也是借助于此方法完成
## writeObject、readObject方法
```JAVA
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();
        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);
        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
```
由于`elementData`不能被序列化，所以源代码中实现了`writeObject`方法将数组中的实际数据写入`ObjectOutputStream `流中
```JAVA
private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;
        // Read in size, and any hidden stuff
        s.defaultReadObject();
        // Read in capacity
        s.readInt(); // ignored
        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            int capacity = calculateCapacity(elementData, size);
            SharedSecrets.getJavaOISAccess().checkArray(s, Object[].class, capacity);
            ensureCapacityInternal(size);
            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
  }
```
在反序列化时，为了保证将实际数据还原到`elementData`中，源代码中实现了`readObject`方法，从`ObjectInputStream `流中将数据还原到`elementData`中
## 总结
>* 由于每次添加新元素都会进行扩容操作(底层其实是数组拷贝，很耗性能)，所以如果开发中知道数据的大小时，最好在创建时就给一个固定容量值
>* ArrayList是线程不安全的，所以不要在多线程中使用，最常见的就是`ConcurrentModificationException`异常，具体的可以查看源码中`iterator`的实现
>* 每次扩容都是为原来的1.5倍，这将导致空间浪费，可以通过`trimToSize`方法将此 ArrayList 实例的容量调整为列表的当前大小
>* 根据源代码中`remove`方法可知ArrayList可以存储null