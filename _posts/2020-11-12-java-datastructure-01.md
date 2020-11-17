---
layout:     post
title:      "东拉西扯数据结构-01"
subtitle:   "表、栈、队列、List、ArrayList、LinkedList"
date:       2020-11-12
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/datastructure-bkg01.jpg"
tags:
    - Java
    - 数据结构
---
> 数据结构与算法分析 java语言描述（原书第3版）-读书笔记

### 表
##### 抽象数据类型（abstract data type，ADT）  
抽象数据类型(abstractdatatype,ADT)是带有一组操作的一些对象的集合。抽象数据类型是数学的抽象;在ADT的定义中没有地方提到关于这组操作是如何实现的任何解释。诸如表、集合、图以及与它们各自的操作一起形成的这些对象都可以被看做是抽象数据类型，这就像整数、实数、布尔数都是数据类型一样。整数、实数和布尔数各自都有与之相关的操作，而抽象数据类型也是如此。对于集合ADT，可以有像添加(add),删除(remove)以及包含(contain)这样一些操作。当然，也可以只要两种操作并(union)和查找(find)，这两种操作又在这个集合上定义了一种不同的ADT  
##### 数组
数组广为人知的特性是：取值快，插入和删除慢。但是，对于插入和删除来说要分情况看待：  
在位置0进行操作：空间复杂度T(n)=O(N)  
在数组的最高端进行操作：空间复杂度T(n)=O(1)  
##### 单链表
链表广为人知的特性是：取值慢，插入和删除快。但是，对于插入和删除来说要分情况看待：  
在表的前端添加项或删除第一项的特殊情形此时也属于常数时间的操作，当然要假设到链表前端的链是存在的。只要我们拥有到链表最后节点的链，那么在链表末尾进行添加操作的特殊情形(即让新的项成为最后一项)可以花费常数时间。因此，典型的链表拥有到该表两端的链。删除最后一项比较复杂，因为必须找出指向最后节点的项，把它的next链改成null，然后再更新持有最后节点的链。在经典的链表中，每个节点均存储到其下一节点的链，而拥有指向最后节点的链并不提供最后节点的前驱节点的任何信息  
##### 双链表
在单链表的基础知识，让每一个节点再持有一个指向它在表中的前驱节点的链，就变成了双链表  
##### Collection和Iterator接口
其实在java中的表最主要是List这个接口，List继承自Collection，所以有必要先理一下Collection的一些东西  

- Collection接口  
Collection接口扩展了Iterable接口，实现Iterable接口的那些类可以使用增强for循环来遍历：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/Iterable-field-method.jpg)
<center>Iterable接口中的方法</center>
其中，forEach方法和spliterator方法是Java8中新增的方法。关于forEach方法中Consumer接口，可以参考[这篇内容：待办项]()

- Iterator接口  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/Iterator-field-method.jpg)  
<center>Iterator的结构图</center>  
实现Iterable接口的集合必须提供一个称为iterator的方法，该方法返回一个Iterator类型的对象。Iterator接口的思路是，通过iterator方法，每个集合均可创建并返回给客户一个实现Iterator接口的对象，并将当前位置的概念在对象内部存储下来  

1. 每次调用Iterator接口的next()方法时，都会集合尚未见到的下一项。这里比较有意思了，第一个元素就是在第一次调用next()方法时得到的  
2. Iterator和Collection都有各自的remove方法，Iterator的remove方法的主要优点在于，而Collection的remove方法必须首先找出要被删除的项才能删除它，多了查找的开销  
3. Concurrent-ModificationException异常：  
这是一个比较有意思的东西，如果在遍历一个集合的同时，外部对它进行了结构更改（add、remove或者clear），那么迭代器很可能会抛出这个异常  
**但是，如果是迭代器自己调用了自己的remove方法时，就不会抛出异常，所以当有必在遍历的同时使用remove的时候，更建议使用Iterator接口的remove方法**  

- ListIterator接口
继承自Iterator接口，增加了previous方法和hasPrevious方法，实现了对表从后往前进行遍历

##### List
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/linkedList-ArrayList.jpg)  
<center>List、ArrayList和LinkedList关系图</center>

- List  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/List-field-method.jpg)
<center>List的接口图</center>

##### ArrayList
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/ArrayList-diagrams.jpg)
<center>ArrayList的继承实现关系</center>

- 成员变量
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/ArrayList-class-filed.jpg)  
<center>成员变量图</center>  
1. 默认容量：**DEFAULT_CAPACITY=10**，并不是说默认生成的就是这么大，在没有指定容量的时候，此时elementData是个空的数组。**当第一次add的时候，就变成默认容量10了**    
2. 最大容量：**MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8**  
3. EMPTY_ELEMENTDATA：当用户指定该 ArrayList 容量为 0 时，返回该空数组`{}`  
4. DEFAULTCAPACITY_EMPTY_ELEMENTDATA：当用户没有指定 ArrayList 的容量时(即调用无参构造函数)，返回的是该空数组`{}`  

- 其他性质  
1. boolean add(E e)方法  
此方法追加数据时，如果原数组是空的，添加1个元素时数组容量变为10；如果原数组不为空，追加元素时容量变为原来的1.5倍。如果扩容后的容量大于分配给ArrayList的容量，判断需要的容量是否比分派的容量大，是就把Integer.MAX_VALUE赋值给minCapacity，否就用MAX_ARRAY_SIZE(Integer.MAX_VALUE - 8)  
2. Arrays.copyof()方法  
底层调用了另一个copyof方法（也就是下面的System.arraycopy()）    
3. System.arraycopy()方法  
该方法被标记了native，调用了系统的C/C++代码，在JDK中是看不到的，但在openJDK中可以看到其源码。该函数实际上最终调用了C语言的memmove()函数，因此它可以保证同一个数组内元素的正确复制和移动，比一般的复制方法的实现效率要高很多，很适合用来批量处理数组。Java强烈推荐在复制大量数组元素时用该方法，以取得更高的效率    
4. 查找给定元素索引值等的方法中，源码都将该元素的值分为null和不为null两种情况处理  
5. **ArrayList中允许元素为null**  
6. arrayList本质上就是一个elementData数组  
7. arrayList区别于数组的地方在于能够自动扩展大小，其中关键的方法就是gorw()方法  
8. arrayList中removeAll(collection c)和clear()的区别就是removeAll可以删除批量指定的元素，而clear是全是删除集合中的元素  

ArrayList详细的介绍可以参考网络上的这篇[博客](https://blog.csdn.net/u010250240/article/details/89762912)  

##### LinkedList  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/LinkedList-diagrams.jpg)
<center>LinkedList的继承实现关系</center>  

LinkedList内部是一个**双向链表**的实现，一个节点除了保持自身的数据外，还持有前，后两个节点的引用。所以就数据存储上来说，它相比使用数组作为底层数据结构的ArrayList来说，会更加耗费空间  
LinkedList没有任何同步手段，所以多线程环境须慎重考虑，可以使用`Collections.synchronizedList(new LinkedList(...));`来保证线程安全  

- 遍历  
虽然LinkedList也提供了get（int index）方法，但是底层中每次调用get（int index）方法的时候，都需要从链表的头部或者尾部进行遍历，每一的遍历时间复杂度是O(index)，所以**不推荐通过get（int index）遍历LinkedList，建议采用迭代器进行遍历**  

```java
//foreach方式
private static void iterateByForEach(LinkedList<Integer> linkedList){
    for (Integer j:linkedList) {
        //TODO
    }
}

//Iterator方式
private static void iterateByIterator(LinkedList<Integer> linkedList){
    Iterator<Integer> iterator = linkedList.iterator();
    while (iterator.hasNext()){
        iterator.next();
    }
}
```

- 类成员  
LinkedList内部有两个引用，一个first，一个last，分别用于指向链表的头和尾，它们的引用类型是Node；另外有一个size，用于标识这个链表的长度  
- 数据结构  
1. Node<E>：LinkedList中的私有内部类，LinkedList中就是通过Node来存储集合中的元素  
2. E item：节点的值  
3. Node<E> next：当前节点的后一个节点的引用（可以理解为指向当前节点的后一个节点的指针）  
4. Node<E> prev：当前节点的前一个节点的引用（可以理解为指向当前节点的前一个节点的指针） 

- 构造方法  
1. public LinkedList()：构造一个空的链表  
2. public LinkedList(Collection<? extends E> c)：调用addAll()方法将集合中的元素添加到链表中  

- List接口的添加操作  
1. add(E e)方法：内部实现是linkLast(E e)方法将元素添加到链表尾部，结果返回boolean   
2. add(int index,E e)：  
检查index的范围，否则抛出异常  
如果插入位置是链表尾部，那么调用linkLast方法   
如果插入位置是链表中间，那么调用linkBefore方法  
3. addAll的两个重载方法  

- Deque接口的添加操作  
1. addFirst(E e)方法：内部通过linkFirst(E e)实现  
2. addLast(E e)方法：linkLast(E e)方法将元素添加到链表尾部  
3. offer(E e)方法：对应List接口添加操作中的add（E e）方法  
4. offerFirst(E e)方法：通过addFirst(E e)方法实现，返回boolean  
5. offerLast(E e)方法（区别同上）  

- 检索操作  
1. E get(int index)获取指定节点数据  
首先判断index在链表的哪边，如果插入位置是链表尾部那么调用linkLast方法；如果插入位置是链表中间，那么调用linkBefore方法；然后遍历查找index或者size-index次，找出对应节点  
2. 获得位置为0的头节点数据  
getFirst()：链表为空时，抛出异常  
element()：使用getFirst()实现，链表为空时，抛出异常  
peek()：链表为空时，返回null  
peekFirst()：链表为空时，返回null  
3. 获得位置为size-1的尾节点数据  
getLast()：链表为空时，抛出异常  
peekLast()：链表为空时，返回null  
4. 根据对象得到索引（找不到则返回-1）  
idnexOf()：从前往后遍历  
lastIndexOf()：从后往前遍历  
5. 检查链表是否包含某对象  
内部调用indexOf()方法  

- 删除操作  
1. 删除指定对象  
一次只会删除一个匹配的对象  
从前向后遍历  
以是否为null做区分，调用unlink()方法  
2. 按照位置删除对象  
删除任意位置的元素：boolean remove(int index)  
删除头节点的对象：
    - 链表为空时，抛出异常：remove()、removeFirst()、pop()  
    - 链表为空时，返回null：poll()、pollFirst()  

删除尾节点的对象：removeLast()、pollLast()

### 栈(Stack)
##### 特点
- 具有LIFO特性的表，常用操作是push（进栈）和pop（出栈）  
- ArrayList和LinkedList都支持栈操作  
- 尾递归：在方法最后一行的递归调用，往往这种递归可以通过while循环来实现，这样可以避免栈溢出  

以Java中的Stack类为例（**实际上也是通过数组去实现的**），记录一下主要方法：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/Stack-diagrams.jpg)

##### boolean empty()
测试栈是否为空  
##### synchronized E peek()
返回堆栈顶部的对象，但不从堆栈中移除它
##### synchronized E pop()
移除堆栈顶部的对象，并返回该对象  
##### E push(E item)
把项压入堆栈顶部，原理是通过将元素追加的数组的末尾中  
##### synchronized int search(Object o)
 查找“元素o”在栈中的位置，由栈底向栈顶方向数

### 队列(Queue)
先入先出（FIFO）的数据结构，常用的操作是入队（enqueue）和出队（dequeue）  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/Queue-diagrams-01.jpg)  
Queue接口与List、Set同一级别，都是继承了Collection接口    
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/Queue-diagrams-02.jpg)
LinkedList实现了Deque接口  

下面以Java中的Queue接口为例，记录一下主要方法：  

##### boolean add(E e)
增加一个元索，如果队列已满，则抛出一个IIIegaISlabEepeplian异常
##### E remove()
移除并返回队列头部的元素，如果队列为空，则抛出一个NoSuchElementException异常
##### E element()
返回队列头部的元素，如果队列为空，则抛出一个NoSuchElementException异常
##### boolean offer(E e)
添加一个元素并返回true，如果队列已满，则返回false
##### E peek()
返回队列头部的元素，如果队列为空，则返回null
##### E poll()
移除并返问队列头部的元素，如果队列为空，则返回null


