---
title: LruCache的简单实现
date: 2018-07-17
tags:
- cache
- LRU
categories:
- algorithm
---
由于前面分析过[LinkedHashMap](https://cqjokers.top/2018/07/12/java/LinkedHashMap/)，知道其内部运用了一个双向链表结构来存储数据，所以能够保证数据的有序性，并且源代码里预留了三个方法，我们可以基于它们来实现一个LRU缓存。
```JAVA
public class LRUHashMap<K,V> extends LinkedHashMap<K, V> {

	private static final long serialVersionUID = 1L;
	//最大缓存数目
	private final int maxCapacity;
	//负载因
	private static final float DEFAULT_LOAD_FACTOR = 0.75f;

	public LRUHashMap(int maxCapacity) {
		super(maxCapacity, DEFAULT_LOAD_FACTOR, true);
		this.maxCapacity = maxCapacity;
	}

	@Override
	protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
		return size() > maxCapacity;
	}
	
}
```
由于`LinkedHashMap`是线程不安全的，所以要想在多线程环境使用可以使用 Collections.synchronizedMap()方法实现线程安全操作