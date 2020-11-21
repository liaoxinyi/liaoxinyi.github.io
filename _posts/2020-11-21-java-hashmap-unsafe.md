---
layout:     post
title:      "HashMap怎么就不安全了？"
subtitle:   "HashMap的并发分析"
date:       2020-11-21
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-bkg02.jpg"
tags:
    - Java
    - 数据结构
---
> 方志朋公众号-读书笔记

### HashMap-JDK1.7-死循环
##### 问题定位
死循环发生在HashMap的扩容函数中，根源在transfer函数，jdk1.7中HashMap的transfer函数如下：

```java
void transfer(Entry[] newTable, boolean rehash) {
     int newCapacity = newTable.length;
     for (Entry<K,V> e : table) {
         while(null != e) {
             Entry<K,V> next = e.next;
             if (rehash) {
                 e.hash = null == e.key ? 0 : hash(e.key);
             }
             int i = indexFor(e.hash, newCapacity);
             e.next = newTable[i];
             newTable[i] = e;
             e = next;
         }
     }
 }
```

在对table进行扩容到newTable后，需要将原来数据转移到newTable中，注意10-12行代码，这里可以看出在转移元素的过程中，使用的是头插法，也就是链表的顺序会翻转，这里也是形成死循环的关键点。下面进行模拟死循环

##### 问题模拟  
- 假设    
1. hash算法为简单的用key mod链表的大小
   
2. 最开始hash表size=2，key=3,7,5，则都在table\[1\]中
   
3. 然后进行resize，使size变成4

- 未resize前的数据结构如下  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-01.jpg)  
- 如果在单线程环境下，最后结果如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-02.jpg)  
- 多线程时  
在多线程环境下，假设有两个线程A和B都在进行put操作。线程A在执行到transfer函数中第11行代码处挂起  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-03.jpg)  
此时线程A中运行结果如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-04.jpg)  
线程A挂起后，此时线程B正常执行，并完成resize操作，结果如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-05.jpg)  

**这里需要特别注意的点：由于线程B已经执行完毕，根据Java内存模型，现在newTable和table中的Entry都是主存中最新值：7.next=3，3.next=null**

此时切换到线程A上，在线程A挂起时内存中值如下：e=3，next=7，newTable\[3\]=null，代码执行过程如下:

```txt
newTable[3]=e ----> newTable[3]=3
e=next ----> e=7
```

此时结果如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-06.jpg)  
继续循环：

```txt
e=7
next=e.next ----> next=3【从主存中取值】
e.next=newTable[3] ----> e.next=3【从主存中取值】
newTable[3]=e ----> newTable[3]=7
e=next ----> e=3
```

结果如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-07.jpg)  
再次进行循环：  
```txt
e=3
next=e.next ----> next=null
e.next=newTable[3] ----> e.next=7 即：3.next=7
newTable[3]=e ----> newTable[3]=3
e=next ----> e=null
```

注意此次循环：e.next=7，而在上次循环中7.next=3，出现环形链表，并且此时e=null循环结束，结果如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-08.jpg)  
在后续操作中只要涉及轮询hashmap的数据结构，就会在这里发生死循环，造成悲剧  

### HashMap-JDK1.7-数据丢失
##### 问题定位
和死循环的定位一样，发生在扩容的时候  
##### 问题模拟
类似前文的模拟，初始时：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-09.jpg)  
线程A和线程B进行put操作，同样线程A挂起：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-10.jpg)  
此时线程A的运行结果如下：
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-11.jpg)  
此时线程B已获得CPU时间片，并完成resize操作：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-12.jpg)  

同样注意由于线程B执行完成，newTable和table都为最新值：5.next=null

此时切换到线程A，在线程A挂起时：e=7，next=5，newTable\[3\]=null

执行newtable\[i\]=e，就将**7放在了table\[3\]**的位置，此时next=5。接着进行下一次循环：  

```txt
e=5
next=e.next ----> next=null，从主存中取值
e.next=newTable[1] ----> e.next=5，从主存中取值
newTable[1]=e ----> newTable[1]=5
e=next ----> e=null
```

将5放置在table\[1\]位置，此时e=null循环结束，3元素丢失，并形成环形链表。并在后续操作hashmap时造成死循环  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/hashmap-unsafe-13.jpg)  

### HashMap-JDK1.8-死循环
在jdk1.8中对HashMap进行了优化，在发生hash碰撞，不再采用头插法方式，而是直接插入链表尾部，因此不会出现环形链表的情况，但是在多线程的情况下仍然不安全  
##### 问题定位
并发时的put方法
##### 问题模拟
如果线程A和线程B同时进行put操作，刚好这两条不同的数据hash值一样，并且该位置数据为null，所以这线程A、B都会进入第6行代码中。假设一种情况，线程A进入后还未进行数据插入时挂起，而线程B正常执行，从而正常插入数据，然后线程A获取CPU时间片，此时线程A不用再进行hash判断了，问题出现：线程A会把线程B插入的数据给覆盖，发生线程不安全  