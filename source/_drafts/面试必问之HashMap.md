---
title: 面试必问之HashMap
tags: HashMap
categories: Java学习笔记
---
# 前言
总结了一个月的面试经验(其实也就3场面试...)，HashMap这个点基本上是必被问道的的，因此想再看一遍HashMap的源码，对其底层加深一下印象。

# 什么是HashMap？
HashMap本质当然是Map，是一个存储Key-Value键值对的数据结构。由于它会对每个Key进行Hash操作存储定位，因此它的查询以及插入效率非常高。在理想状态下时间复杂度为O(1)，最差情况下时间复杂度为O(n)。但同时它也是线程不安全的，如果想要使用线程安全的HashMap建议使用ConcurrentHashMap。

# HashMap的内部结构
HashMap的本质是一个链表+数组的数据结构。（JDK1.8还增加了红黑树的结构） 其源码如下：

```java
    //主要存储数据的对象 table
    transient Node<K,V>[] table;

    //Node类
    static class Node<K,V> implements Map.Entry<K,V> {
    //算出的hash值，用来定义数组索引的位置
    final int hash;
    //key值
    final K key;
    //value值
    V value;
    //指向下一个Node对象
    Node<K,V> next;

    //get set方法
    ...
  }    
```
从源码我们可以看到Node类作为HashMap的静态内部类，本身实现了Map.Entry,它是真正用来存储键值对的，同时他也是一个链表。而HashMap内部维护了一个Node数组名为table的变量，也被称为哈希桶数组，是HashMap的核心，各项操作都围绕它展开。

HashMap还有几个比较重要的成员变量，在了解其原理之前我们必须先认识他们。
```java

//HashMap目前保存键值对数量(也就是大小)
transient int size;

//记录HashMap内部结构发生变化的次数
transient int modCount;

//所能容纳的键值对极限
int threshold;

//负载因子
final float loadFactor;
```


size和其他容器的size一样，代表着目前HashMap存放的键值对数量

threshold代表HashMap能容纳键值对的个数极限，如果一旦超过这个极限HashMap就会进行相应的扩容。

loadFactor是负载因子，它也会影响到threhold的变化：threshold = length * loadFactor 。这里的
length也就是table数组的大小 ，默认值给的是16，值得注意的是它的值永远是2的n次方，这是HashMap做的一个优化点，也是面试经常问的，后续会讲明其原因。从上面这个公式可以看出，loadFactor越小，threshold也会越小，会导致扩容频率变高，容易造成空间浪费。loadFactor越大，threshold也会越大，扩容频率变小，但会增大Hash冲突的机率，耗时更久(这一块目前不理解没关系，后面会讲解)。因此loadFactor的大小填写，是需要我们再时间复杂度与空间复杂度之间权衡的，系统默认值给的是0.75，符合大部分情况，一般不建议修改。

modCount字段主要用来记录HashMap内部结构发生变化的次数，主要用于迭代的快速失败。强调一点，内部结构发生变化指的是结构发生变化，例如put新键值对，但是某个key对应的value值被覆盖不属于结构变化。(这一部分其实我目前也不是非常的了解，后续详解吧)
