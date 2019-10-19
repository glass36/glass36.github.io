---
title: 探究JDK1.8中HashMap的实现原理
tags:
  - HashMap
  - Java
categories: Java学习笔记
date: 2019-10-19 16:39:33
---

>本博文碍于作者的学识与见解，难免会有疏漏错误之处，请谅解。<br/>
转载请注明出处: [www.sshenzx.com](www.sshenzx.com) 谢谢~



# 前言
总结了一个月的面试经验(其实也就3场面试...)，HashMap这个点基本上是必被问到的，因此想再看一遍HashMap的源码，对其底层加深一下印象。作为我的第一篇正式博客同时也知道没几个人会看这个博客，我就不说什么欢迎指正的客套话了，爱咋咋地。

(注：本文所述的HashMap以及代码都是基于JDK1.8的，与之前的版本区别较大，需要注意)

# 什么是HashMap？
HashMap本质当然是Map，是一个存储Key-Value键值对的数据结构。由于它会对每个Key进行hash运算存储其定位，因此它的查询以及插入效率非常高。在理想状态下时间复杂度为O(1)，最差情况下时间复杂度为O(n)。但同时它也是线程不安全的，如果想要使用线程安全的HashMap建议使用ConcurrentHashMap。
<!-- more -->
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
从源码我们可以看到Node类作为HashMap的静态内部类，本身实现了Map.Entry,它是真正用来存储键值对的，同时他也是一个链表。而HashMap内部维护了一个Node数组名为table的变量，也被称为哈希桶，是HashMap的核心，各项操作都围绕它展开。他的整个内部图如下：

![](https://raw.githubusercontent.com/glass36/BlogFiles/master/images/HashMap_5.jpg)


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

**size**
和其他容器的size一样，代表着目前HashMap存放的键值对数量

**threshold**
代表HashMap能容纳键值对的个数极限，如果一旦超过这个极限HashMap就会进行相应的扩容。

**loadFactor**
是负载因子，它也会影响到threhold的变化：threshold = length * loadFactor 。这里的
**length**
也就是table数组的大小 ，默认值给的是16，值得注意的是它的值永远是2的n次方，这是HashMap做的一个优化点，也是面试经常问的，后续会讲明其原因。从上面这个公式可以看出，loadFactor越小，threshold也会越小，会导致扩容频率变高，容易造成空间浪费。loadFactor越大，threshold也会越大，扩容频率变小，但会增大Hash冲突的机率，耗时更久(这一块目前不理解没关系，后面会讲解)。因此loadFactor的大小填写，是需要我们再时间复杂度与空间复杂度之间权衡的，系统默认值给的是0.75，符合大部分情况，一般不建议修改。

**modCount**
字段主要用来记录HashMap内部结构发生变化的次数，主要用于迭代的快速失败。强调一点，内部结构发生变化指的是结构发生变化，例如put新键值对时modCount会加一，但是某个key对应的value值被覆盖不属于结构变化所以modCount不会变化。

# HashMap 源码解析
对HashMap有一定的了解后，我们终于可以开始手撕源码啦。我们首先就从put方法进入开始看源码

```java
public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
}

/**
 * Implements Map.put and related methods
 *
 * @param hash key的hash值
 * @param key  key值
 * @param value value值
 * @param onlyIfAbsent 如果key对应的value已经存在，是否覆盖value值
 * @param evict 如果为false表示处于创建状态
 * @return 返回key对应的上一个value数据，如果没有就返回null
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
                 ...
}
```
我们可以看到put方法本身内部又调用了putVal，它有五个入参，在注解上我也已经写了，
它们分别代表：
 1. hash：key的hash值
 2. key： key值
 3. value: value值
 4. onlyIfAbsent：如果key对应的value已经存在，是否覆盖value值
 5. evict：如果为false表示处于创建状态，仅仅为LinkedHashMap方便排序操作，HashMap中可以忽略

针对于onlyIfAbsent为true，很明显我们的HashMap如果key已经对应存在对应value映射，在put时会覆盖掉原来的值，想不覆盖的话可以调用putIfAbsent()方法。

有一个重点其实是在方法hash(key)上，这个方法计算出了key的hash值，他是HashMap的精华所在，也是面试中常问的一个考点。接下来我们将重点说明下这个方法以及如何使用这个hash值
## hash()方法以及确定哈希桶数组索引位置
hash()方法其实顾名思义就是用来获取key的hash的一个hash值的,但是HashMap里的hash()方法似乎与一般的直接key.hashCode()不太一样，我们先看看它到底是什么样的神奇操作。
其源码如下:
```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
歪果仁的代码追求精简，虽然就两行，但是看起来还是蛮恶心的，但是还是可以看出来如果key为空，那么hash值就默认问0，这也就说明HashMap的key是可以为空的，而我们也知道HashTable的key不能为空。如果不为空则会执行后面一长串，这一长串可以分步骤看
1. h = key.hashCode() 获取了key的hashCode值
2. h^h>>>16   将hashCode的值异或他的高16位获取到hash值

我们知道hashCode本身是一个32位的int类型，进行这样的操作就等于将hashCode的高16位异或它的低16位得到一个新的hash值。

![](https://raw.githubusercontent.com/glass36/BlogFiles/master/images/HashMap_1.jpg)

但是拿到这样一个hash值的作用是什么呢？我们可以先想一下如何利用key的hash值确定每个key的哈希桶索引位置而且还需要尽量均衡。第一个想到的当然是用hash值对哈希桶的长度(length)进行取模的操作。(当然我本人是连这种方式都想不到的= =)即:

 **index = hash % length**


这种方式可以用随机的hash值算出随机的索引并且分配也尽量均匀。没错！!HashMap也是这么想的。但是这种取模运算本身是对CPU运算开销比较大的，为了优化速度，HashMap采取了更优雅的方式，在putVal的核心代码里可以看到这么一段：

```java
/**
*为了方便理解,代码有改动+精简,但是整体思路没变
*/
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {            
                    //table就是HashMap的table即哈希桶
                    Node<K,V>[] tab = table;    
                    //哈希桶的长度
                    int length = tab.length;  
                    //确定索引的位置
                    int index = (length-1) & hash;       
                    //索引的值
                    Node<K,V> p = tab[index];
               }
```

我们可以看到HashMap采用了hash值"与"length-1的方式来确定索引位置。即：

 **index = hash & length-1**

还记得之前我们说过hashTable的length总是2的n次方吗？这个是故意为了加快取模运算而设计的。当length的长度为2的n次方时hash值对length取模或者"与"上length-1得到的效果是一样的，但是"与"操作相比取模速度能快很多。
而之所以hash值要用拿hashcode的高16位异或低16位的原因根据官方的解释是从速度、功效、质量来考虑的，这么做可以在数组table的length比较小的时候，也能保证考虑到高低Bit都参与到Hash的计算中，同时不会有太大的开销。

![](https://raw.githubusercontent.com/glass36/BlogFiles/master/images/HashMap_2.jpg)

<font color=red><b>小Tips:</b>

hash()方法中的高16位异或16位的计算方式，是在JDK1.8之后才加上的，在JDK1.7及之前的版本里是indexFor()方法，直接用hashCode&length-1计算出索引位置。如果面试官有问HashMap的JDK1.7与JDK1.8的区别可不要忘记说了哦
</font>

## putVal()方法解析
了解了如何确定哈希桶的索引地址后我们终于可以来看看核心操作方法putVal了，不多说直接撕源码
```java
/**
 * Implements Map.put and related methods
 *
 * @param hash key的hash值
 * @param key  key值
 * @param value value值
 * @param onlyIfAbsent 如果key对应的value已经存在，是否覆盖value值
 * @param evict 如果为false表示处于创建状态
 * @return 返回key对应的上一个value数据，如果没有就返回null
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;

    //如果发现哈希桶为空或者长度为0，则进行扩容
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

        //确定哈希桶的索引位置，如果里面目前没有数据，则直接插入新Node进去
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);

        //如果有数据，即发生hash冲突
    else {
        Node<K,V> e; K k;

        //如果新插入数据的key与发生冲突的Node的key值相同，则记录下来
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;

            //如果冲突的Node已经是红黑树结构了，则调用树的插入方法
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

            //如果是链表则开始遍历链表
        else {
            for (int binCount = 0; ; ++binCount) {
                //整个链表遍历完成后仍未发现key值相同的数据,则在链表结尾插入数据
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    //插入数据后，发现链表数量已经到达一定长度后链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                //发现新插入的key值与链表已存在的数据key值相同，则记录下来
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        //如果有替换原有数据，则将新值替换旧值并返回旧值
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            //如果onlyIfAbsent为true就不替换了
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            //方便linkedHashMap排序操作，HashMap里可无视
            afterNodeAccess(e);
            return oldValue;
        }
    }
    //modCount+1
    ++modCount;
    //如果新加如元素后发现键值对超过所能容纳的键值对极限则进行扩容
    if (++size > threshold)
        resize();        
      //方便linkedHashMap排序操作，HashMap里可无视    
    afterNodeInsertion(evict);
    return null;
}

```
根据代码我们可以大致理出putVal的操作流程
1. 判断哈希桶大小，如果为空则进行第一次的扩容
2. 计算索引位置index，如果哈希桶索引位置table[index]没有数据则插入数据新数据，并判断键值对数量是否超过了threshHold，超过了则进行扩容。
3. 如果table[index]有数据即发生hash冲突，判断table[index]的key值是否与插入的数据相同，如果相同则使用新的数据覆盖原有数据并返回原有数据，否则继续。
4. 判断table[index]是否为为红黑树，如果是的话则直接在红黑树中插入数据，否则则继续
5. 遍历链表tabe[i],如果存在key相等的数据则覆盖原有数据，并返回原有数据。如果不存在key相等的数据则在链表末尾插入新数据，同时判断链表长度是否到达可以转换红黑树的标准(默认长度为8)，到达的话则进行转换。

具体流程如图如下所示：

![](https://raw.githubusercontent.com/glass36/BlogFiles/master/images/HashMap_3.jpg)


<font color=red><b>小Tips:</b>

链表长度大于8转换成红黑树的操作是在JDK1.8中新加的，目的是当出现大量Hash冲突时也能使用红黑树让查找的时间复杂度由O(n)降低为O(logn)。至于红黑树里面添加数据的源码我就不手撕了，确实比较复杂，我撕不动...知道红黑树是一颗特殊的二叉平衡查找树即可。
</font>

## resize()方法解析
在前面的介绍里我们可以注意到，在插入新数据以及首次插入数据时，HashMap都进行了一次判断是否需要扩容。我们知道Java中数组的长度是固定的，当我们需要扩容时，肯定需要用一个新的数组来替换老的数组，并将数据迁移过去。而这些操作在HashMap中全部体现在resize()方法中:

```java
final Node<K,V>[] resize() {
    //----------------------以下为第一部分，负责初始化定义新容器大小与threshold-----------------------------------
    //获取旧的table，旧的threshhold，旧的长度
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
       //如果容器容量已经到达最大值，则不再扩容了，threshhold也设置为最大值
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        //定义了新容器大小为旧容器大小的两倍，新threshold也为旧threshhold两倍
        //注意这里采用了 << 1操作即地位补0的方式，与 *2 的效果一样且速度更快
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    //如果hashtable尚未定义，则初始化定义容量以及threshold
    else if (oldThr > 0)
        newCap = oldThr;
    else {               
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
       //生成新的newTab来替换之前的oldTable
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
   //-------------------------------------------------------------------------------------------------------
   //-----------------------------以下为第二部分，负责将oldTab的数据导入到newTab中------------------------------
    if (oldTab != null) {
        //遍历oldTab
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                  //如果oldTab[j]只有一个Node，那么直接重新计算index，将Node放入
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                  //如果oldTab[j]已经转换为一个红黑树，则对红黑树进行rehash
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);

                    //对于链表进行操作
                else {
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        //计算链表oldTab[j]新的索引值，如果索引不需要更改，则由loHead链表保存它们
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        //计算链表oldTab[j]新的索引值，如果索引需要更改，则由hiHead链表保存它们
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    //循环结束。如果不需要改变索引位置的Node则直接将loHead保存在newTab[j]中
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    //如果需要改变索引位置的Node则直接将hiHead保存在newTab[j]中
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
扩容代码是非常精巧的，虽然看起来比较复杂，但是如果去细究的话就会发现里面用了很聪明的优化方式。我们可以将代码分为两部分去看(已用注解区分)

**第一部分：初始化新容器**

前面我们说了，扩容肯定是需要用一个新的哈希桶来替换旧的哈希桶的。那么HashMap里面由于为了优化取模速度(上文已提及)，让哈希桶的长度必须为2的n次方，所以每一次扩容只需要将原有哈希桶的大小扩充一倍即可。于是我们就可以看到newCap = oldCap << 1，这种通过位偏移的方式相对于使用newCap = oldCap * 2的方式效率会更高，也算设计团队的一个优化地方。同时我们在代码里也能看到，如果哈希桶的容器仍未定义即第一次使用时，也会进行数据初始化，这一部分很简单就不再叙述了。

**第二部分：将旧哈希桶的数据导入到新哈希桶中**

这一部分讲道理思路其实也很简单。在JDK1.7中的实现方式是遍历旧哈希桶中的所有Node(rehash)，重新计算它们的索引位置，依次放入到新的哈希桶中。(值得注意的是JDK1.7中旧链表迁移新链表的时候，如果在新表的数组索引位置相同，则链表元素会倒置，感兴趣的去看JDK1.7的源码就能明白其中意思了)而在JDK1.8中，HashMap采用了一种更聪明的方式。由于哈希桶的长度一定是2的n次方，所以对Node的重新计算索引也就是hash值对length-1重新取模的索引值，要么是原索引的值，要么是原索引+原length的值。可能比较绕，看下图就明白了，n为table的长度，图(a)表示扩容前的key1和key2两种key确定索引位置的示例，图(b)表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希与高位运算结果。

![](https://raw.githubusercontent.com/glass36/BlogFiles/master/images/HashMap_4.jpg)

我们会发现确定索引值是否要变化的关键，其实在于hash值新增的那一位bit，如果那一位为0，则新的索引位置就是原索引位置，如果那一位为1，则新的索引位置就是原索引位置+oldLength。所以我们重新计算索引时根本不需要重新计算它们的位置，只需要判断新增的那一位bit是0还是1就可以了，大大优化了效率。因此我们再源码中可以看到对 (e.hash & oldCap) == 0 进行了判断，如果为true代表索引为原位置，如果为false代表索引为原位置+oldLength。同时我们也可以发现在代码段中定义了loHead与hiHead两个链表，一个是存放不用变更索引位置的Node，一个是存放变更索引位置的Node。定义这两个链表使得变更新索引位置的链表不会像JDK1.7一样倒置。

## get()，remove()等方法的思路
至此我们已经基本了解完了HashMap的put方法的流程。当然HashMap也提供了get()，remove()等方法，但是大致思路与put是基本一致的。都是要通过计算key的hash值确定，哈希桶的位置，然后判断是否有哈希冲突的情况，如果有的话再判断是链表还是红黑树，最终确定到对应的值。具体的流程都是一样的，再次就不赘述了，可以自己跟着源码看下去。

# HashMap的线程安全问题
开始的时候我们就已经提到HashMap是线程不安全的，那么为什么说HashMap是线程不安全的呢？ 在JDK1.7中，resize时更新到新的索引位置时采用的是头插法的方法，会让链表导致。但是在多线程的情况下头插法会出现链表回环的情况(解释比较复杂，就不详细说明了)。在JDK1.8中，我们已经取消了这种方式，但仍然会出现线程不安全的情况。如putVal中：

```java
/**
 * 已省略大量代码
 */
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
     ...
     ...
     //确定哈希桶的索引位置，如果里面目前没有数据，则直接插入新Node进去
     if ((p = tab[i = (n - 1) & hash]) == null)
     tab[i] = newNode(hash, key, value, null);
     ...
     ...
}

```

在putVal中如果线程A与线程B同时进入了该方法，同时判断了(p = tab[i = (n - 1) & hash]) == null 为true，这样会导致其中一个插入的数据会被丢失的情况出现。

其实HashMap使用了公用的变量Node[]table进行各类复杂的操作，是肯定会出现线程不安全的情况的，所以使用的时候一定要注意。当然也不是没有解决方案，最早方法自然就是使用HashTable，但是众所周知HashTable的效率很低，它对每个操作都加了锁。这里推荐使用java.util.concurrent包下的ConcurrentHashMap，它使用了分段锁的方式细化锁的粒度，在保证了线程安全的同时也不比HashMap慢多少(ConcurrentHashMap也是面试常问的点，后续有时间也一定会专门写一篇对其原理的分析)。

# 小结
那在最后来总结下我个人认为的HashMap的精髓
1. HashMap的是一个链表+数组+红黑树的存储结构
2. 确定哈希桶位置使用了 hash & length-1 的方式显著提高了效率
3. JDK1.8中，链表长度超过8将会自动转换为红黑树，提高了查找效率
4. HashMap是线程不安全的，想要线程安全请使用ConcurrentHashMap

<br/>
<br/>

**小结后的小小结：**

终于完成了我的第一篇正式博文。突然发现写博客真的是一个很累的事情啊！！！为了这篇博文，看源码看资料写博文这一系列操作花费了我大概有10个小时，不过对我自己也有了一个很大的提升，不管怎么说算是一个好的开始？哈哈哈，希望能坚持下去吧。over！

# 参考资料

[美团技术博客——Java 8系列之重新认识HashMap](https://zhuanlan.zhihu.com/p/21673805) [图片素材来源]

[漫画：高并发下的HashMap](https://www.cnblogs.com/qingyunzong/p/9143249.html)
