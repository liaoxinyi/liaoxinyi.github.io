---
layout:     post
title:      "Running还是Runnable？"
subtitle:   "Java线程状态Runnable详解"
date:       2020-10-12
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-thread-running.jpg"
tags:
    - Java
    - 多线程
---
> 原始资料来源于[程序猿DD：为什么 Java 线程没有Running状态？](https://mp.weixin.qq.com/s?src=11&timestamp=1603758592&ver=2669&signature=E3sH2BNnAev9o4aAmRV*H*akRQXQyEKQ4FMz9AxYzZe74jBApx6LH7IT7NkTbKMUP2przZC9B*mxfvhq8srDTpO*VClnHdII0nBZd7di-QeGZnWUHeEMNw*DN9TLb5LC&new=1)

### 前言
&emsp;&emsp;为什么要记录一下这个东西呢？按理来说，Java的线程状态其实早就烂大街，谁不知道有那6种状态呢？其实，也就是来自于 Thread 类下的 State 这一内部枚举类中所定义的状态。但是，在学习线程的一开始我经常看到这样的一幅总结图：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-thread-life1.png)
<center>Java线程的生命周期</center>  
其中，Runnable里面细分了Ready和Running，这就让我觉得很疑惑了。那Runnable和Ready的意思有啥区别吗？而且为啥Running要放在Runnable里面呢？后来，在DD大佬的文章中，找到了相关的介绍，而且比较详细，所以根据文章的内容，我大致记录一下自己的认识
### Runnable和Ready
##### Ready更多属于传统线程级别
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-thread-running-traditional.jpg)
<center>传统线程（进程）状态</center>
##### Runnable包含了传统Ready
一个在 JVM 中执行的线程就可以视作是Runnable状态，处于 runnable 状态下的线程正在 Java 虚拟机中执行，但它可能正在等待 来自于操作系统的其它资源，比如处理器  
所以，**对Java线程状态而言，不存在所谓的running状态，它的 runnable 状态包含了 running状态**

### CPU的线程轮转
>现在的时分多任务操作系统架构通常都是用所谓的“时间分片 ”方式进行抢占式 轮转调度，更复杂的可能还会加入优先级（priority）的机制

上面引用自文章中的语句，其中，涉及到多个概念：
1. 时分：time-sharing  
2. 多任务：multi-task  
3. 时间分片：time quantum or time slice  
这个时间分片通常是很小的，可以理解为一个线程在CPU上工作的时间（有点像蹲坑时间），一个线程一次最多只能在 cpu 上运行比如10-20ms 的时间（**此时处于 Running 状态**），也即大概只有0.01秒这一量级，时间片用后就要被切换下来放入调度队列的末尾等待再次调度。（**也即回到 ready 状态**），底层的本质是利用定时器定时中断来驱动  
**如果期间进行了 I/O 的操作还会导致提前释放时间分片，并进入等待队列。又或者是时间分片没有用完就被抢占，这时也是回到 ready 状态**  
线程在被拉入CPU和被切出CPU这一过程，也叫做上下文切换（context switch），这一过程中CPU会把被切出去的线程的执行状态保存到内存中以便后续的恢复执行。另外上下文切换的频率其实是挺高的，而且由于切换的时间非常非常短，所以给人的假象就是在线程在“并发”进行。时间分片也是可配置的，如果不追求在多个线程间很快的响应，也可以把这个时间配置得大一点，以减少切换带来的开销。如果是多核CPU，才有可能实现真正意义上的并发，这种情况通常也叫并行 （pararell）  
4. 抢占式轮转调度：preemptive round-robin  

所以，因为上下文切换的时间短，频率高，而java里面的线程往往是服务与监控的（这一点不是很懂是什么意思），所以Running和ready其实就没必要区分那么细了。现今主流的 JVM 实现都把 Java 线程一一映射到操作系统底层的线程上，把调度委托给了操作系统，我们在虚拟机层面看到的状态实质是对底层状态的映射及包装。JVM 本身没有做什么实质的调度，把底层的 ready 及 running 状态映射上来也没多大意义，因此，统一成为runnable 状态是不错的选择  

### 当I/O阻塞时
##### 传统的I/O阻塞时
一旦线程中执行到 I/O 有关的代码，相应线程立马被切走，然后调度 ready 队列中另一个线程来运行。为啥要这样做，因为和I/O的速度比起来，CPU的执行效率简直不要太高。这让我想起来了为啥在服务中都尽量减少I/O，也就是缓存的作用为啥这么大  
虽然I/O的线程被切走了，但是不会被放到调度队列中去，因为很可能再次调度到它时，I/O 可能仍没有完成。相应的，线程会被放到所谓的等待队列中，此时也叫做Waiting，也就是常说的，这个线程被阻塞了  

- **中断驱动（interrupt-driven）**
而当 I/O 完成时，则用一种叫中断 （interrupt）的机制来通知 cpu。这也是控制反转 （IoC）机制的一种体现，cpu不用反复去询问硬盘，硬盘对 cpu 说：”别老来问我 IO 做完了没有，完了我自然会通知你的“，这个描述我觉得挺形象的  
cpu 收到一个来自硬盘的中断信号，并进入中断处理例程，手头正在执行的线程因此被打断，回到 ready 队列。而先前因 I/O 而waiting 的线程随着 I/O 的完成也再次回到 ready 队列，这时 cpu 可能会选择它来执行。这里为啥说是可能，因为并不一定说你这个线程I/O一结束后回到ready队列中就马上轮到你来执行  
##### Java的I/O阻塞时
在Java中，阻塞式I/O时，其实此时的线程也是Runnable状态的（对应的是Ready状态）。这里牵引出另外一个概念：阻塞式，比如阻塞式方法，阻塞式I/O等等。虽然叫阻塞，但是此时的线程却是Runnable的，有点神奇  
>进行传统上的 IO 操作时，口语上我们也会说“阻塞”，但这个“阻塞”与线程的 BLOCKED 状态是两码事

### Runnable的真实面目
如果要真正探讨线程的Runnable状态的话，需要分清JVM的层次和操作系统的层次。为什么这么说，因为当进行阻塞式的 IO 操作时，底层的操作系统线程确实处在阻塞状态，但是此时JVM 认为线程还在执行。因为这个时候只是说 cpu 不执行线程了，但网卡可能还在监听，所以JVM不会理会底层的CPU单独是不是阻塞了，它认为线程还在执行。而操作系统的线程状态是围绕着 cpu 这一核心去述说的，这与 JVM 的侧重点是有所不同的  
**换句话说：JVM中的RUNNABLE 状态对应了传统的ready，running以及部分的waiting 状态**  
