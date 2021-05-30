---
layout:     post
title:      "东拉西扯数据结构-03"
subtitle:   "HashSet、LinkedHashSet、TreeSet、Map、HashMap、ConcurrentHashMap、并发容器"
date:       2020-11-20
author:     "ThreeJin"
header-mask: 0.7
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-bkg04.jpg"
tags:
    - Java
    - 数据结构
    - 多线程
---
> 《数据结构与算法分析 java语言描述》（原书第3版）+《愚公要移山》-读书笔记+网络总结（think123等）

### 前言
过关斩将，终于到了老生常谈的各种集合了，集合旗下的各类结构简直多得很，大体上分为两个类别：单列集合（`Collection接口`）和双列结合（`Map接口`）

### Collection接口
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection01.jpg)  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection02.jpg)  
### List接口、Queue接口
List旗下的内容包括`Vector`、`ArrayList`和`LinkedList`等，这块内容可以参考前面，`Queue`也可以直接参考前面[电梯](https://www.threejinqiqi.fun/2020/11/12/java-datastructure-01/)  
### Set接口
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection03.jpg)  
- 元素不重复  
- 没索引  
- 存和取的顺序不一致

### HashSet类
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection04.jpg)  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection05.jpg)  
##### 特性
- 元素不重复  
- 不保证集合中元素的顺序  
- 允许包含值为null的元素，但最多只能一个  
- **底层使用的是HashMap的key或者LinkedHashMap的key来保存所有元素**  

##### 底层
- 相关 HashSet 的操作，基本上都是直接调用底层 `HashMap`或者`LinkedHashMap`的相关方法来完成  
- 需要提前为将要保存到 HashSet 中的对象覆盖 `hashCode()` 和 `equals()`  

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
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection06.jpg)  
继承自HashSet类，底层靠的是`LinkedHashMap`

**<font color=red>关键，可以保证怎么存就怎么取</font>**

#### TreeSet类
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection07.jpg)  

##### 成员变量
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection08.jpg)  

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
`m.put(e, PRESENT)`一看e这个key已经存在了，就会将PRESENT替换掉相应的value值。然后map返回这个value。value不等于null，于是返回false。很明显插入失败了。如果把PRESENT替换成null呢？这时候`m.put(e, PRESENT)`一看e这个key已经存在了，于是map返回null。这时候null==null。整个add方法返回的就是true，这就矛盾了，所以这里使用了PRESENT这个对象就能有效地避免这种情况  

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

### Map接口
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection09.jpg)  
### HashMap类
以JDK1.8为例进行分析  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection10.jpg)  
##### 内部类  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection11.jpg)  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection12.jpg)  

- HashMap最早是在jdk1.2中开始出现的，一直到jdk1.7一直没有太大的变化。但是到了jdk1.8突然进行了一个很大的改动：
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection14.jpg)  
<center>HashMap在JDK7和8中的数据结构变化</center>  
之前jdk1.7的存储结构是数组+链表，到了jdk1.8变成了数组+链表+红黑树  
- jdk8中在**数组大于等于64且链表节点数大于等于8**的时候转换为红黑树。当**红黑树节点数量小于6**时又转换为链表  
- HashMap在JDK8中是一个`Node<K,V>数组`，JDK7中是`Entry数组`，**数组长度默认为16**  
- Node可能是链表结构，也可能是红黑树结构，Node对象中包含了键和值，其中next也是一个Node对象，它就是用来处理hash冲突的，形成一个链表

##### 成员变量  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection13.jpg)  
 
- **initialCapacity初始容量**  
要输入一个2的N次幂的值，即使输入的不是符合要求的值，虚拟机会根据输入的值，找一个离20最近的2的N次幂的值  
- **loadFactor负载因子**  
负载因子，默认值是0.75。负载因子表示一个散列表的空间的使用程度，有这样一个公式：initailCapacity\*loadFactor=HashMap的容量。所以负载因子越大则散列表的装填程度越高，也就是能容纳更多的元素，元素多了，链表大了，所以此时索引效率就会降低。反之，负载因子越小则链表中的数据量就越稀疏，此时会对空间造成烂费，但是此时索引效率高  
**这里有个很有意思的事情，为什么负载因子默认是0.75呢？**  
这是经过试验很多次后得到的这个值，此时空间利用率比较高，而且避免了相当多的Hash冲突，使得底层的链表或者是红黑树的高度比较低，提升了空间效率  
如果负载因子很大，比如1，也就意味着，只有当数组全部填充了，才会发生扩容。这就带来了很大的问题，因为Hash冲突是避免不了的。在数组从未满到满的过程中可能已经出现了大量的Hash冲突，底层的红黑树变得异常复杂。对于查询效率极其不利。如果给定合适的负载因子，可以让数组未满时就扩容，牺牲了时间来保证空间的利用率  
如果负载因子比较小，比如负载因子是0.5的时候，这也就意味着，当数组中的元素达到了一半就开始扩容，既然填充的元素少了，Hash冲突也会减少，那么底层的链表长度或者是红黑树的高度就会降低，查询效率就会增加。一句话总结就是负载因子太小，虽然时间效率提升了，但是空间利用率降低了    
- **构造HashMap的时候并没有初始化数组容量，而是在第一次put元素的时候才进行初始化的**

##### hash值  
HashMap中主要是通过key的hashCode来计算hash值的，只要hashCode相同，计算出来的hash值就一样  

```java
static final int hash(Object key) {
     int h;
     return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

- **通过key的hashcode与16异或计算**，之所以选择异或计算，是因为这样计算出来的hash比较均匀（0和1的概率大致相当，都为50%），结合后面用h&(length-1)的方法确定数组下标，这样不容易出现冲突  
- 在JDK7版本中进行了四次扰动运算，即无符号右移了4次，但在JDK8版本中只进行了一次扰动运算

不容易出现冲突不等于不会出现冲突，所以就需要解决hash值冲突：**开放定址法（发生冲突，继续寻找下一块未被占用的存储地址），再散列函数法，链地址法**

HashMap即是采用了链地址法解决hash值冲突，也就是数组+链表的方式，如下图：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection17.jpg)  
<center>HashMap数组+链表的示意图</center>  
基本思想是将所有哈希地址为i的元素构成一个称为同义词链的单链表，并将单链表的头指针存入哈希表的第i个单元中，因而查找、插入和删除主要在同义词链中进行。链地址法适用于经常进行插入和删除的情况

##### put(K key, V value)方法  

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

可以看到底层其实是调用了`putVal`方法

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection15.jpg)  
<center>put方法的流程</center>  

- 通过`hash`函数计算key的hash值，然后进入putVal方法  
- 如果hash表为空，调用`resize()`方法创建一个hash表  
- 根据hash值索引hash表中`Node数组`中对应下表的位置，判断该位置是否有hash碰撞  
    - 没有碰撞，直接插入映射入hash表  
    - 有碰撞，遍历桶中节点  
        - 第一个节点匹配，记录该节点  
        - 第一个节点没有匹配，桶中结构为红黑树结构，按照红黑树结构添加数据，记录返回值  
        - 第一个节点没有匹配，桶中结构是链表结构。遍历链表，如果找到key映射节点，记录，退出循环，如果没有找到则在链表尾部添加节点。插入后判断链表长度是否大于转换为红黑树要求，满足则转为红黑树结构  
        - 用于记录的值判断是否为null，不为则是需要插入的映射key在hash表中原来有，替换值，返回旧值putValue方法结束

总结如下：  
- **put方法的末尾：如果map的容量(存储元素的数量)大于阈值则进行扩容，扩容为之前容量的2倍**
- `Hashtable`对哈希表的散列是用hash值对数组length取模（即除法散列法），因为会用到除法运算，效率低，HashMap中则通过`h&(length-1)`的方法来代替取模，这是改进  
- **数组大小一定要是2的整数次幂**  
    - 数组length为2的整数次幂的话，`h&(length-1)`就相当于对length取模，这样相比除法效率更高  
    - 此时`length-1`为奇数，奇数的最后一位是1，这样便保证了`h&(length-1)`的最后一位可能为0，也可能为1（这取决于h的值），即与后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性，而如果length为奇数的话，很明显length-1为偶数，它的最后一位是0，这样`h&(length-1)`的最后一位肯定为0，即只能为偶数，这样任何hash值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间，增加了碰撞的几率，减慢了查询的效率，因此，length取2的整数次幂，是为了使不同hash值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列  
- **发生冲突时节点插入链表的链头还是链尾呢？**  
JDK7是头插法，JDK8采用尾插法  
- 和 Java7 稍微有点不一样的地方就是，Java7 是先扩容后插入新值的，Java8 先插值再扩容  
- put的时候通过装载因子来判定空闲槽位还有多少，如果超过装载因子的值就会动态扩容,HashMap会扩容为原来的两倍大小(初始容量为16,即槽(数组)的大小为16)

##### 扩容的流程  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection16.jpg)  
<center>扩容的流程</center>  
HaspMap扩容就是就是先计算新的hash表容量和新的容量阀值，然后初始化一个新的hash表，将旧的键值对重新映射在新的hash表里。如果在旧的hash表里涉及到红黑树，那么在映射到新的hash表中还涉及到红黑树的拆分
##### get(Object key)方法 
get()的方法就相对来说要简单一些了，它最重要的就是找到key是存放在哪个位置：  
- 先根据key计算hash值  
- 然后`(n-1) & hash`确定元素在数组中的位置  
- 判断数组中第一个元素是否是需要找的元素  
- 节点如果是树节点,则在红黑树中寻找元素，否则就在链表中寻找对应的节点

##### Fail-Fast机制
成员变量：**transient int modCount**  
- modCount声明为`volatile`，保证线程之间修改的可见性  
- 对HashMap内容的修改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的`expectedModCount`，在迭代过程中，判断modCount跟expectedModCount是否相等，如果不相等就表示已经有其他线程修改了Map

##### 遍历

##### 线程不安全
[电梯](https://www.threejinqiqi.fun/2020/11/12/java-hashmap-unsafe/)
##### 怎么确保线程安全
使用ConcurrentHashMap

### ConcurrentHashMap类

##### JDK1.7的实现
在JDK1.7版本中，ConcurrentHashMap的数据结构是由一个`Segment数组`和`多个HashEntry`组成，如下图所示：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection20.jpg)

```java
static class  Segment<K,V> extends  ReentrantLock implements  Serializable {
}
```

主要实现原理是实现了锁分离的思路解决了多线程的安全问题，Segment数组的意义就是将一个大的table分割成多个小的table来进行加锁，也就是上面的提到的锁分离技术，而每一个Segment元素存储的是HashEntry数组+链表，这个和HashMap的数据存储结构一样

- 初始化  

1. ConcurrentHashMap的初始化是会通过位与运算来初始化Segment的大小，默认为16，最多65536个  
2. 每一个Segment元素下的HashEntry的初始化也是按照位与运算来计算

- put操作  
    - 需要两次Hash才能到达指定的HashEntry，第一次hash到达`Segment`,第二次到达Segment里面的Entry,然后在遍历entry链表  
    - 从上Segment的继承体系可以看出，Segment继承了`ReentrantLock`,也就带有锁的功能  
    - 当执行put操作时，会进行第一次key的hash来定位Segment的位置，如果该Segment还没有初始化，即通过CAS操作进行赋值，然后进行第二次hash操作，找到相应的HashEntry的位置  
    - 找到相应的HashEntry的位置后，会通过继承`ReentrantLock`的`tryLock()`方法尝试去获取锁，如果获取成功就直接插入相应的位置（链表的尾端），如果已经有线程获取该`Segment`的锁，那当前线程会以自旋的方式去继续的调用`tryLock()`方法去获取锁，超过指定次数就挂起，等待唤醒

- get操作  
ConcurrentHashMap的get操作跟HashMap类似，只是ConcurrentHashMap第一次需要经过一次hash定位到Segment的位置，然后再hash定位到指定的HashEntry，遍历该HashEntry下的链表进行对比，成功就返回，不成功就返回null

- **size操作**  
计算ConcurrentHashMap的元素大小是一个有趣的问题，因为一般用到这个map时都会是并发操作的，也就是在计算size的时候，往往还在并发的插入数据，要解决这个问题，JDK7用两种方案：

1. 方案一：  
使用不加锁的模式去尝试多次计算ConcurrentHashMap的size，最多三次，比较每次计算的结果，结果一致就认为当前没有元素加入，计算的结果是准确的  
2. 方案二：  
如果第一种方案不符合，就会给每个Segment加上锁，然后计算ConcurrentHashMap的size返回  

##### JDK1.8的实现
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-collection19.jpg)  
- **已经摒弃了Segment的概念，而是直接用Node数组+链表+红黑树的数据结构来实现，整个看起来就像是优化过且线程安全的HashMap**。虽然在JDK1.8中还能看到Segment的数据结构，但是已经简化了属性，只是为了兼容旧版本  
- **并发控制使用Synchronized和CAS来操作**  
- **Node**  
1. ConcurrentHashMap存储结构的基本单元，类似于HashMap中的Node，用于存储数据  
2. 是一个链表，但是只允许对数据进行查找，不允许进行修改  

- TreeNode  
1. 继承自Node，但是数据结构换成了红黑树的数据的存储结构，用于存储数据  
2. 当链表的节点数大于8时会转换成红黑树的结构，他就是通过TreeNode作为存储结构代替Node来转换成黑红树  

- TreeBin  
1. 存储树形结构的容器，或者说封装TreeNode的容器  
2. 提供转换黑红树的一些条件和锁的控制  

- 初始化
1. ConcurrentHashMap的初始化其实是一个空实现，并没有做任何事  
2. 默认的初始化操作并不是在构造函数实现的，而是在put操作中实现  
3. 其他构造函数则参考HashMap即可

- **put方法**  
1. 如果没有初始化就先调用`initTable()`方法来进行初始化过程  
2. **如果没有hash冲突就直接CAS插入**  
3. 如果还在进行扩容操作就先进行扩容  
4. **如果存在hash冲突，就加锁(Synchronized将该链表锁定)来保证线程安全**，这里有两种情况，一种是链表形式就直接遍历到尾端插入，一种是红黑树就按照红黑树结构插入   
5. 如果添加成功就调用`addCount()`方法统计size，并且检查是否需要扩容  
6. 在强一致的场景中`ConcurrentHashMap`就不适用，原因是**ConcurrentHashMap 中的 `get、size `等方法没有用到锁**，因此返回的数据就不准确, ConcurrentHashMap 是弱一致性的，因此有可能会导致某次读无法马上获取到写入的数据

- 扩容方法：transfer()  
有helpTransfer()方法的调用多个工作线程一起帮助进行扩容  
- get方法  
1. 计算hash值，定位到该table索引位置，如果是首节点符合就返回  
2. 如果遇到扩容的时候，会调用标志正在扩容节点ForwardingNode的find方法，查找该节点，匹配就返回  
3. 以上都不符合的话，就往下遍历节点，匹配就返回，否则最后就返回null

- 总结  
1. 可以看出JDK1.8版本的`ConcurrentHashMap`的数据结构已经接近`HashMap`，相对而言，`ConcurrentHashMap`只是增加了同步的操作来控制并发  
2. 从JDK1.7版本的**ReentrantLock+Segment+HashEntry**，到JDK1.8版本中**synchronized+CAS+HashEntry+红黑树**  
3. JDK1.8的实现降低了锁的粒度：**JDK1.7版本锁的粒度是基于Segment的**，包含多个HashEntry，而JDK1.8锁的粒度就是Node，但保留了简化属性的Segment接口以兼容旧版本  
4. JDK1.8版本使用红黑树来优化链表，数据结构变得更加简单，使得操作也更加清晰流畅  
5. **为啥使用`synchronized`而不用`ReentrantLock`？**  
    - 因为粒度降低了，在相对而言的低粒度加锁方式，synchronized并不比ReentrantLock差。在粗粒度加锁中ReentrantLock的优势是可以通过Condition来控制各个低粒度的边界，更加的灵活。而在低粒度中，Condition的优势就没有了  
    - JVM的开发团队从来都没有放弃synchronized，而且基于JVM的synchronized优化空间更大，使用内嵌的关键字比使用API更加自然  
    - 在大量的数据操作下，对于JVM的内存压力，基于API的ReentrantLock会开销更多的内存，虽然不是瓶颈，但是也是一个选择依据  
    
### 并发容器
##### 并发下的Map
- **线程安全，安全的Map对象 `Hashtable`、`ConcurrentHashMap` 以及`ConcurrentSkipListMap` 三个容器**  
- `Hashtable` VS `ConcurrentHashMap`  
    - Hashtable在它所有的获取或者修改数据的方法上都添加了Synchronized  
    - 虽然 ConcurrentHashMap 的整体性能要优于 Hashtable，但在某些场景中，ConcurrentHashMap 依然不能代替 Hashtable  
    - **在强一致的场景中ConcurrentHashMap 就不适用，原因是 ConcurrentHashMap 中的 get、size 等方法没有用到锁，因此返回的数据就不准确**, ConcurrentHashMap 是弱一致性的，因此有可能会导致某次读无法马上获取到写入的数据  
- `ConcurrentHashMap` VS `ConcurrentSkipListMap`  
    - ConcurrentHashMap 数据量比较大的时候，链表会转换为红黑树。红黑树在并发情况下，删除和插入过程中有个平衡的过程，会牵涉到大量节点，因此竞争锁资源的代价相对比较高  
    - ConcurrentSkipListMap由于是基于跳表实现的，它需要锁住的节点要少一些，在高并发场景下性能也要好一些：**新增节点和链接索引都是基于 CAS 操作实现**  
- 总结  
    - 当不需要知道集合中准确数据的时候使用`ConcurrentHashMap`，当需要知道集合中准确数据个数时，则需要用到`HashTable`  
    - 如果数据量特别大，且存在大量增删改查操作，则可以考虑使用`ConcurrentSkipListMap`  
    - **这三个线程安全的容器类`key,value`都不允许为null**,而平时使用的HashMap的key是可以为null,value不能为null的
    
##### 并发下的List
- **线程安全，安全的List对象 `Vector`、`CopyOnWriteList`**  
- `CopyOnWriteList`实现了读操作无锁，写操作则通过先复制一个新的数组，将数据写入到新的数组中，再将新的数组赋值给旧数组来实现，是一种读写分离的并发策略  
- `CopyOnWriteList`更适用于读远大于写的操作，同时业务场景对写入数据的实时获取并没有要求,只需要保证最终能获取到写入数组中的数据就可以了

##### 并发下的Set
- **线程安全，安全的List对象 `CopyOnWriteArraySet`、`ConcurrentSkipListSet`**  
- `CopyOnWriteArraySet`是使用`CopyOnWriteArrayList`实现的，而 `ConcurrentSkipListSet` 又是使用 `ConcurrentSkipListMap`实现的  

##### 并发下的Queue
Queue的分类大致上可以从两个维度来分：  
- 阻塞与非阻塞  
    - 所谓阻塞指的是当队列已满时，入队操作阻塞；当队列已空时，出队操作阻塞  
    - Java 并发包里阻塞队列都用 `Blocking` 关键字标识  
- 单端与双端  
    - 单端指的是只能队尾入队，队首出队；而双端指的是队首队尾皆可入队出队  
    - 单端队列使用 `Queue` 标识，双端队列使用 `Deque` 标识