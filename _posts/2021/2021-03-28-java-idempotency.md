---
layout:     post
title:      "如何保证对外接口的幂等性？"
subtitle:   "如何保证对外接口的幂等性？"
date:       2021-03-28
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/mybatis-bk-01.jpg"
tags:
    - Java

---
> 资料来源于网络上各位前辈（苏三说技术等）

### 前言
什么是幂等性？比如这样的场景：在填写某些form表单时，保存按钮不小心快速点了两次，后台的表中竟然产生了两条重复的数据，只是id不一样。或者，在项目中为了解决接口超时问题，通常会引入了重试机制。第一次请求接口超时了，请求方没能及时获取返回结果（但是此时有可能已经成功了），为了避免返回错误的结果，于是会对该请求重试几次，这样也会产生重复的数据。再比如：mq消费者在读取消息时，有时候会读取到重复消息，如果处理不好，也会产生重复的数据。所以，这里就有一个接口幂等性的概念：  
**接口幂等性是指用户对于同一操作发起的一次请求或者多次请求的结果是一致的，不会因为多次点击而产生了副作用**  
**防重设计和幂等设计其实是有区别的，防重设计主要为了避免产生重复数据，对接口返回没有太多要求。而幂等设计除了避免产生重复数据之外，还要求每次请求都返回一样的结果**
### insert前先select
通常情况下，在保存数据的接口中，我们为了防止产生重复数据，一般会在`insert`前，先根据`name`或`code`字段`select`一下数据。如果该数据已存在，则执行`update`操作，如果不存在，才执行`insert`操作  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/ide-01-01.jpg)  
<center>insert前先select</center>  
上面的方案可能是单体里面用得最多的，成本也低，但是不适合并发的场景
### 悲观锁
通过给数据库加悲观锁，将当前请求的那行数据锁住，在同一时刻只允许一个请求获得锁，更新数据，其他的请求则等待，比如：  
`select * from user id=123 for update;`  
具体的执行流程如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/ide-01-02.jpg)  
<center>悲观锁避免数据重复</center>  
悲观锁需要在同一个事务操作过程中锁住一行数据，如果事务耗时比较长，会造成大量的请求等待，影响接口性能。此外，每次请求接口很难保证都有相同的返回值，所以不适合幂等性设计场景，但是在防重场景中是可以的使用的。在这里顺便说一下，防重设计 和 幂等设计，其实是有区别的。防重设计主要为了避免产生重复数据，对接口返回没有太多要求。而幂等设计除了避免产生重复数据之外，还要求每次请求都返回一样的结果
### 乐观锁
在表中增加一个`timestamp`或者`version`字段，这里以`version`字段为例  
在更新数据之前先查询一下数据：  
`select id,amount,version from user id=123;`  
如果数据存在，假设查到的`version`等于1，再使用`id`和`version`字段作为查询条件更新数据：  
`update user set amount=amount+100,version=version+1 where id=123 and version=1;`  
更新数据的同时`version+1`，然后**判断本次update操作的影响行数，如果大于0，则说明本次更新成功，如果等于0，则说明本次更新没有让数据变更**  
由于第一次请求`version`等于1是可以成功的，操作成功后version变成2了。这时如果并发的请求过来，再执行相同的sql：  
`update user set amount=amount+100,version=version+1 where id=123 and version=1;`  
该update操作不会真正更新数据，最终sql的执行结果影响行数是0，因为version已经变成2了，where中的version=1肯定无法满足条件。  
但为了保证接口幂等性，接口可以直接返回成功，因为version值已经修改了，那么前面必定已经成功过一次，后面都是重复的请求。具体流程如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/ide-01-03.jpg)  
<center>乐观锁保证幂等性</center>  
总结起来的流程就是：  
- 先根据id查询用户信息，包含version字段  
- 根据id和version字段值作为where条件的参数，更新用户信息，同时version+1  
- 判断操作影响行数，如果影响1行，则说明是一次请求，可以做其他数据操作  
- 如果影响0行，说明是重复请求，则直接返回成功

### 唯一索引
绝大数情况下，为了防止重复数据的产生，我们都会在表中加唯一索引，这是一个非常简单，并且有效的方案  
`alter table `order` add UNIQUE KEY `un_code` (`code`);`  
加了唯一索引之后，第一次请求数据可以插入成功。但后面的相同请求，插入数据时会报`Duplicate entry '002' for key 'order.un_code'`异常，表示唯一索引有冲突。虽说抛异常对数据来说没有影响，不会造成错误数据。但是为了保证接口幂等性，我们需要对该异常进行捕获，然后返回成功。  
如果是java程序需要捕获：`DuplicateKeyException`异常，如果使用了spring框架还需要捕获：`MySQLIntegrityConstraintViolationException`异常
### 防重表
有时候表中并非所有的场景都不允许产生重复的数据，只有某些特定场景才不允许。这时候，直接在表中加唯一索引，显然是不太合适的，所以，这个时候`防重表`就应运而生了，该表可以只包含两个字段：主键`id` 和 `唯一索引`，唯一索引可以是多个字段比如：`name`、`code`等组合起来的唯一标识，例如：`susan_0001`  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/ide-01-04.jpg)  
<center>防重表保证幂等性</center>  
**<font color=red>防重表和业务表必须在同一个数据库中，并且操作要在同一个事务中</font>**  
### 分布式锁
其实前面介绍过的加唯一索引或者加防重表，本质是使用了数据库的分布式锁，也属于分布式锁的一种。但由于数据库分布式锁的性能不太好，我们可以改用：`redis`或`zookeeper`，具体流程如下：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/ide-01-05.jpg)  
<center>redis分布式锁保证幂等性</center>  
### token
该方案跟之前的所有方案都有点不一样，需要两次请求才能完成一次业务操作。
第一次请求：客户端请求服务，服务端先生成token存入redis并设置过期时间，然后返回token  
第一次请求：客户端带上token请求服务，服务端根据token在redis中的存在情况来决定是否是重复请求
### 总结
总结后发现，其实幂等性（或者说防重性）和分布式锁的场景是非常类似的，大体上都是通过保证数据的唯一性来进行设计的。