---
layout:     post
title:      "别再乱画时序图了"
subtitle:   "时序图的介绍与绘制"
date:       2019-10-23
author:     "ThreeJin"
header-mask: 0.6
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/idea.jpg"
tags:
    - 工具
---
> 资料学习自[SuperMan-zhang](https://blog.csdn.net/fly_zxy/article/details/80911942)

### 前言
最近在写文档的时候，发现以前画的时序图都是错的，所以在网上找了些资料，把这块遗忘的知识又拾回来一下。时序图(Sequence Diagram)，又名序列图、循序图，是一种UML交互图。它通过描述对象之间发送消息的时间顺序显示多个对象之间的动态协作
### 时序图的元素
在画时序图时会涉及7种元素：角色(Actor)、对象(Object)、生命线(LifeLine)、控制焦点(Activation)、消息(Message)、自关联消息、组合片段。其中前6种是比较常用和重要的元素，剩余的一种组合片段元素不是很常用，但是比较复杂
#### 角色(Actor)
系统角色，可以是人或者其他系统，子系统。以一个小人图标表示
#### 对象(Object)
对象位于时序图的顶部,以一个矩形表示。对象的命名方式一般有三种：
1. 对象名和类名。例如：华为手机:手机、loginServiceObject:LoginService  
2. 只显示类名，不显示对象，即为一个匿名类。例如：:手机、:LoginSservice  
3. 只显示对象名，不显示类名。例如：华为手机:、loginServiceObject  

#### 生命线(LifeLine)
时序图中每个对象和底部中心都有一条垂直的虚线，这就是对象的生命线(对象的时间线)。以一条垂直的虚线表
#### 控制焦点(Activation)
控制焦点代表时序图中在对象时间线上某段时期执行的操作。以一个很窄的矩形表示
#### 消息(Message)
表现代表对象之间发送的信息。消息分为三种类型
- **同步消息(Synchronous Message)**  
消息的发送者把控制传递给消息的接收者，然后停止活动，等待消息的接收者放弃或者返回控制。用来表示同步的意义。以一条实线+实心箭头表示
- **异步消息(Asynchronous Message)**  
消息发送者通过消息把信号传递给消息的接收者，然后继续自己的活动，不等待接受者返回消息或者控制。异步消息的接收者和发送者是并发工作的。以一条实线+大于号表示
- **返回消息(Return Message)**  
返回消息表示从过程调用返回。以小于号+虚线表示

#### 自关联消息
表示方法的自身调用或者一个对象内的一个方法调用另外一个方法。以一个半闭合的长方形+下方实心剪头表示
### 举例说明
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tool-element-example.bmp)
<center>时序图元素说明举例</center>

### 组合片段
组合片段用来解决交互执行的条件和方式，它允许在序列图中直接表示逻辑组件，用于通过指定条件或子进程的应用区域，为任何生命线的任何部分定义特殊条件和子进程。组合片段共有13种，名称及含义如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tool-combine.png)
<center>组合片段说明</center>
####  抉择（Alt）
抉择在任何场合下只发生一个序列。 可以在每个片段中设置一个临界来指示该片段可以运行的条件。else 的临界指示其他任何临界都不为 True 时应运行的片段。如果所有临界都为 False 并且没有 else，则不执行任何片段。Alt片段组合可以理解为if..else if...else条件语句  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tool-chose-index.png)
<center>抉择说明</center>
#### 选项（Opt）
包含一个可能发生或不发生的序列。Opt相当于if..语句  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tool-option-index.png)
<center>选项说明</center>
#### 循环（Loop）
片段重复一定次数，可以在临界中指示片段重复的条件。Loop相当于for语句  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tool-repeat-index.png)
<center>重复说明</center>
#### 并行（Par）
并行处理，片段中的事件可以并行交错。Par相当于多线程  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/tool-parallel-index.png)
<center>并行说明</center>