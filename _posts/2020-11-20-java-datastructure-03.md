---
layout:     post
title:      "东拉西扯数据结构-03"
subtitle:   "HashSet、LinkedHashSet、TreeSet"
date:       2020-11-20
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-bkg02.jpg"
tags:
    - Java
    - 数据结构
---
> 《数据结构与算法分析 java语言描述》（原书第3版）+《愚公要移山》-读书笔记

### 前言
过关斩将，终于到了老生常谈的各种集合了，集合旗下的各类结构简直多得很，大体上分为两个类别：单列集合和双列结合

### 单列集合的根接口-Collection
【01】
【02】
### List根接口和Queue根接口
List旗下的内容包括Vector、ArrayList和LinkedList等，这块内容可以参考前面，Queue也可以直接参考前面[电梯](https://www.threejinqiqi.fun/2020/11/12/java-datastructure-01/)  
### Set根接口
【03】
- 元素不重复  
- 没索引  
- 存和取的顺序不一致

### HashSet类
【04】
【05】
##### 特性
- 元素不重复  
- 不保证集合中元素的顺序  
- 允许包含值为null的元素，但最多只能一个  
- **底层使用的是HashMap的key或者LinkedHashMap的key来保存所有元素**  

##### 底层
- 相关 HashSet 的操作，基本上都是直接调用底层 HashMap或者LinkedHashMap的相关方法来完成  
- 需要提前为将要保存到 HashSet 中的对象覆盖 hashCode() 和 equals()  

##### 构造方法  
- HashSet()  
构造一个新的空 set，其底层 HashMap 实例的默认初始容量是 16，加载因子是 0.75  
- HashSet(Collection<? extends E> c)  
构造一个包含指定 collection 中的元素的新 set  
- HashSet(int initialCapacity)  
构造一个新的空 set，其底层 HashMap 实例具有指定的初始容量和默认的加载因子（0.75）  
- HashSet(int initialCapacity, float loadFactor)  
构造一个新的空 set，其底层 HashMap 实例具有指定的初始容量和指定的加载因子  
- HashSet(int initialCapacity, float loadFactor, boolean dummy)  
构造一个新的空 set，其底层 LinkedHashMap 

##### 常用方法  
- size()  
返回此 set 中的元素的数量（set 的容量）  
- isEmpty()  
如果此 set 不包含任何元素，则返回 true  
- add(E e)  
如果此 set 中尚未包含指定元素，则添加指定元素  
- iterator()  
返回对此 set 中元素进行迭代的迭代器  
- remove(Object o)  
如果指定元素存在于此 set 中，则将其移除  
- contains(Object o)  
如果此 set 包含指定元素，则返回 true  
- clear()  
从此 set 中移除所有元素  
          
##### 总结
- 实现原理，基于哈希表（hashmap）实现  
- 不允许重复键存在，但可以有null值  
- 哈希表存储是无序的  
- 添加元素时把元素当作hashmap的key存储，HashMap的value是存储的一个固定值object  
- 排除重复元素是通过equals检查对象是否相同  
- 判断2个对象是否相同，先根据2个对象的hashcode比较是否相等（如果两个对象的hashcode相同，它们也不一定是同一个对象，如果不同，那一定不是同一个对象）如果不同，则两个对象不是同一个对象，如果相同，在将2个对象进行equals检查来判断是否相同，如果相同则是同一个对象，不同则不是同一个对象  
- 如果要完全判断自定义对象是否有重复值，这个时候需要将自定义对象重写对象所在类的hashcode和equals方法来解决  
- 哈希表的存储结构就是：数组+链表，数组的每个元素都是以链表的形式存储的  

#### LinkedHashSet类
【06】
**继承自HashSet类，底层靠的是LinkedHashMap，也就是HashMap+双向链表**

**所以，可以保证怎么存就怎么取**

#### TreeSet类
【07】

##### 成员变量
【08】

**TreeSet只使用了NavigableMap的key，这个和HashSet只是用HashMap或者LinkedHashMap的key一样**  
为什么不再将null作为NavigableMap的value呢？那样还节省内存。这里需要看一下TreeSet的增删方法：  
```java
//add方法其实就是Map的put方法
public boolean add(E e) {
    return m.put(e, PRESENT)==null;
}
//remove方法其实就是Map的remove方法
public boolean remove(Object o) {
    return m.remove(o)==PRESENT;
}
```
在add方法中，我们会出现两种put情况：  
1. 元素e不重复  
直接插入即可  
2. 元素e重复  
m.put(e, PRESENT)一看e这个key已经存在了，就会将PRESENT替换掉相应的value值。然后map返回这个value。value不等于null，于是返回false。很明显插入失败了。如果把PRESENT替换成null呢？这时候m.put(e, PRESENT)一看e这个key已经存在了，于是map返回null。这时候null==null。整个add方法返回的就是true，这就矛盾了，所以这里使用了PRESENT这个对象就能有效地避免这种情况  

##### 特点  
- 保存无重复的数据  
- 对元素进行了排序  
- 底层通过红黑树实现  
- 非线程安全  
- java8新增分割器spliterator() 方法  
- **TreeSet的元素存储在NavigableMap中的key中**  

##### TreeSet和HashSet怎么选择  
- HashSet是用Hash表来存储数据，而TreeSet是用二叉树来存储数据  
- 在不需要排序的时候，还是建议优先使用HashSet，因为速度更快，二叉树需要排序就免不了跳转旋转，所以速度会很慢  

### 双列集合的根接口-Map
【09】
### HashMap-JDK1.8
【10】
##### 内部类  
【11】
【12】

1. HashMap最早是在jdk1.2中开始出现的，一直到jdk1.7一直没有太大的变化。但是到了jdk1.8突然进行了一个很大的改动：  
【14】

**之前jdk1.7的存储结构是数组+链表，到了jdk1.8变成了数组+链表+红黑树**  

2. HashMap其实就是一个Node<K,V>数组（JDK7中是Entry数组），**数组长度默认为16（原因参考为啥Node数组的容量一定要是2的整数次幂）**，这个Node可能是链表结构，也可能是红黑树结构，Node对象中包含了键和值，其中next也是一个Node对象，它就是用来处理hash冲突的，形成一个链表  

3. 如果插入的元素key的hashcode值相同（这会导致生成的hash值一样，就冲突啦），那么这些key也会被定位到Node数组的同一个格子里，如果不超过8个使用链表存储。如果**链表的长度超过8个且数组的长度不小于64**，会调用treeifyBin函数，将链表转换为红黑树。那么即使所有key的hashcode完全相同，由于红黑树的特点，查找某个特定元素，也只需要O（logn）的开销，贼快！！！

**一般在这个时候，往往早就已经扩容了**

##### 成员变量  
【13】
 
- initialCapacity初始容量  
要输入一个2的N次幂的值，及时输入的不是符合要求的值，虚拟机会根据输入的值，找一个离20最近的2的N次幂的值  
- **loadFactor负载因子**  
负载因子，默认值是0.75。负载因子表示一个散列表的空间的使用程度，有这样一个公式：initailCapacity\*loadFactor=HashMap的容量。所以负载因子越大则散列表的装填程度越高，也就是能容纳更多的元素，元素多了，链表大了，所以此时索引效率就会降低。反之，负载因子越小则链表中的数据量就越稀疏，此时会对空间造成烂费，但是此时索引效率高  
**这里有个很有意思的事情，为什么负载因子默认是0.75呢？**  
这是经过试验很多次后得到的这个值，此时空间利用率比较高，而且避免了相当多的Hash冲突，使得底层的链表或者是红黑树的高度比较低，提升了空间效率  
如果负载因子很大，比如1，也就意味着，只有当数组全部填充了，才会发生扩容。这就带来了很大的问题，因为Hash冲突是避免不了的。在数组从未满到满的过程中可能已经出现了大量的Hash冲突，底层的红黑树变得异常复杂。对于查询效率极其不利。如果给定合适的负载因子，可以让数组未满时就扩容，牺牲了时间来保证空间的利用率  
如果负载因子比较小，比如负载因子是0.5的时候，这也就意味着，当数组中的元素达到了一半就开始扩容，既然填充的元素少了，Hash冲突也会减少，那么底层的链表长度或者是红黑树的高度就会降低，查询效率就会增加。一句话总结就是负载因子太小，虽然时间效率提升了，但是空间利用率降低了    

##### hash值  
HashMap中主要是通过key的hashCode来计算hash值的，只要hashCode相同，计算出来的hash值就一样  

```java
static final int hash(Object key) {
     int h;
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

**通过key的hashcode与16异或计算**，之所以选择异或计算，是因为这样计算出来的hash比较均匀（0和1的概率大致相当，都为50%），结合后面用h&(length-1)的方法确定数组下标，这样不容易出现冲突

不容易出现冲突不等于不会出现冲突，所以就需要解决hash值冲突：**开放定址法（发生冲突，继续寻找下一块未被占用的存储地址），再散列函数法，链地址法**

HashMap即是采用了链地址法解决hash值冲突，也就是数组+链表的方式，如下图：  
【17】
基本思想是将所有哈希地址为i的元素构成一个称为同义词链的单链表，并将单链表的头指针存入哈希表的第i个单元中，因而查找、插入和删除主要在同义词链中进行。链地址法适用于经常进行插入和删除的情况

##### put(K key, V value)方法  
【15】

- Hashtable对哈希表的散列是用hash值对数组length取模（即除法散列法），因为会用到除法运算，效率低，HashMap中则通过h&(length-1)的方法来代替取模，这是改进  
- 哈希表的容量一定要是2的整数次幂  
首先，数组length为2的整数次幂的话，h&(length-1)就相当于对length取模，这样便保证了散列的均匀，同时也提升了效率；其次，length为2的整数次幂的话，为偶数，这样length-1为奇数，奇数的最后一位是1，这样便保证了h&(length-1)的最后一位可能为0，也可能为1（这取决于h的值），即与后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性，而如果length为奇数的话，很明显length-1为偶数，它的最后一位是0，这样h&(length-1)的最后一位肯定为0，即只能为偶数，这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间，增加了碰撞的几率，减慢了查询的效率，因此，length取2的整数次幂，是为了使不同hash值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列  
- 发生冲突时节点插入链表的链头还是链尾呢？  
JDK7是头插法，JDK8采用尾插法  
- 和 Java7 稍微有点不一样的地方就是，Java7 是先扩容后插入新值的，Java8 先插值再扩容

##### 扩容的流程  
【16】

HaspMap扩容就是就是先计算 新的hash表容量和新的容量阀值，然后初始化一个新的hash表，将旧的键值对重新映射在新的hash表里。如果在旧的hash表里涉及到红黑树，那么在映射到新的hash表中还涉及到红黑树的拆分
##### Fail-Fast机制
成员变量：**transient int modCount**  
- modCount声明为volatile，保证线程之间修改的可见性  
- 对HashMap内容的修改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的expectedModCount，在迭代过程中，判断modCount跟expectedModCount是否相等，如果不相等就表示已经有其他线程修改了Map

##### 遍历

##### 线程不安全
[电梯](https://www.threejinqiqi.fun/2020/11/12/java-hashmap-unsafe/)
##### 怎么确保线程安全
使用ConcurrentHashMap

### ConcurrentHashMap-JDK1.8

datastructure-collection19


##### 总结
- JDK7中  
【18】

数据结构是由一个Segment数组和多个HashEntry组成，主要实现原理是实现了锁分离的思路解决了多线程的安全问题

Segment数组的意义就是将一个大的table分割成多个小的table来进行加锁，也就是上面的提到的锁分离技术，而每一个Segment元素存储的是HashEntry数组+链表，这个和HashMap的数据存储结构一样

ConcurrentHashMap 与HashMap和Hashtable 最大的不同在于：put和 get 两次Hash到达指定的HashEntry，第一次hash到达Segment,第二次到达Segment里面的Entry,然后在遍历entry链表

- JDK8中 
【19】

已经摒弃了Segment的概念，而是直接用Node数组+链表+红黑树的数据结构来实现，并发控制使用Synchronized和CAS来操作，整个看起来就像是优化过且线程安全的HashMap，虽然在JDK1.8中还能看到Segment的数据结构，但是已经简化了属性，只是为了兼容旧版本

- 对比  
1. 可以看出JDK1.8版本的ConcurrentHashMap的数据结构已经接近HashMap，相对而言，ConcurrentHashMap只是增加了同步的操作来控制并发

2. 从JDK1.7版本的**ReentrantLock+Segment+HashEntry**，到JDK1.8版本中**synchronized+CAS+HashEntry+红黑树**

3. JDK1.8的实现降低了锁的粒度  
JDK1.7版本锁的粒度是基于Segment的，包含多个HashEntry，而JDK1.8锁的粒度就是HashEntry（首节点）

4. JDK1.8版本的数据结构变得更加简单，使得操作也更加清晰流畅  
因为已经使用synchronized来进行同步，所以不需要分段锁的概念，也就不需要Segment这种处理方式了（但保留了简化属性的Segment接口以兼容旧版本），由于粒度的降低，实现的复杂度也增加了

5. JDK1.8使用红黑树来优化链表  
基于长度很长的链表的遍历是一个很漫长的过程，而红黑树的遍历效率是很快的，代替一定阈值的链表，这样形成一个最佳拍档

6. 为啥使用synchronized而不用ReentrantLock？  
    - 因为粒度降低了，在相对而言的低粒度加锁方式，synchronized并不比ReentrantLock差。在粗粒度加锁中ReentrantLock的优势是可以通过Condition来控制各个低粒度的边界，更加的灵活。而在低粒度中，Condition的优势就没有了  
    - JVM的开发团队从来都没有放弃synchronized，而且基于JVM的synchronized优化空间更大，使用内嵌的关键字比使用API更加自然  
    - 在大量的数据操作下，对于JVM的内存压力，基于API的ReentrantLock会开销更多的内存，虽然不是瓶颈，但是也是一个选择依据  