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
<!--more-->
* 自己实现一个LRU缓存
```JAVA
public class LRUCache<K,V> {
	//缓存最大数目
	private int maxCacheSize;
	//头节点
	private Node<K,V> head;
	//尾节点
	private Node<K, V> tail;
	private final Map<K, Node> cacheMap = new HashMap<K, Node>();
	public LRUCache(int maxCacheSize) {
		this.maxCacheSize = maxCacheSize;
	}
	
	public V get(K key) {
		if(cacheMap.containsKey(key)) {
			Node node = cacheMap.get(key);
			removeNode(node);
			addNode(node);
			return (V) node.value;
		}
		return null;
	}
	
	public void put(K key,V value) {
		//判断是否已经存在
		if(cacheMap.containsKey(key)) {
			//移动到首部
			Node node = cacheMap.get(key);
			removeNode(node);
			addNode(node);
		} else {
			//添加节点到首部
			Node<K, V> node = new Node<K, V>(key, value);
			cacheMap.put(key, node);
			addNode(node);
		}
	}
	
	public int size() {
		return cacheMap.size();
	}
	
	private void addNode(Node node) {
		if(cacheMap.size() > maxCacheSize) {
			cacheMap.remove(tail.key);
			//删除尾节点
			removeNode(tail);
		}
		node.before = null;
		Node h = head;
		head = node;
		if(h == null)
			tail = node;
		else {
			node.after = h;
			h.before = node;
		}
	}
	
	/**
	 * 移除节点
	 * @param node
	 */
	private void removeNode(Node node) {
		if(node.after == null)
			tail = node.before;
		else
			node.after.before = node.before;
		
		if(node.before == null) 
			head = node.after;
		else
			node.before.after = node.after;
	}
	
	@Override
	public String toString() {
		StringBuilder sb = new StringBuilder() ;
        Node<K,V> node = tail ;
        while (node != null){
            sb.append(node.key).append(":")
                    .append(node.value)
                    .append("-->") ;
            node = node.before ;
        }
        return sb.toString();
	}
	
	private class Node<K,V> {
		
		K key;
		
		V value;
		
		Node<K,V> before,after;
		
		public Node(K key,V value) {
			this.key = key;
			this.value = value;
		}
	}
}
```
测试效果如下
```JAVA
@Test
public void test() {
    LRUCache<Integer, Integer> cache = new LRUCache<Integer, Integer>(3);
    cache.put(1, 1);
    cache.put(2, 2);
    cache.put(3, 3);
    cache.put(4, 4);
    System.out.println(cache);
    
    cache.get(3);
    cache.put(5, 5);
    System.out.println(cache);
	
    cache.put(6, 6);
    System.out.println(cache);
}
2:2-->3:3-->4:4-->
4:4-->3:3-->5:5-->
3:3-->5:5-->6:6-->
```
* 内部使用的是一个HashMap存入数据，不过值是一个Node节点
* 使用了一个双向链表将各数据关联起来
* 由于存入的值是进行转换后的Node节点，所以在查询的时候，相比遍历整个链表效率要好一点
* 由于内部使用的是HashMap所以所有操作都是线程不安全的。