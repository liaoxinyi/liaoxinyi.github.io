---
layout:     post
title:      "春天一样的代码长啥样？Spring-01"
subtitle:   "Transactional注解"
date:       2021-04-07
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-redis-bk.png"
tags:
    - MySQL

---
> 资料来源于网络上各位前辈（wind瑞_等）

### 前言
有时候真想学会NARUTO的分身术，因为总感觉时间不够用。最近开始将之前在项目中研究的一些东西沉淀时，发现时间完全不够用。前面才开了一个Redis的坑，这里又开了Spring的坑，也不知道啥时候能够填的起来，但无论怎样，还是话不多说，直接开搞。
### @Transactional是如何工作的？
最开始使用`@Transactional`注解的时候，是因为在SpringBoot中使用了JPA，发现查询的时候正常但是保存或者修改东西的时候，无法生效。百度一波后，结果告诉我要在方法上添加@Transactional注解，然后就依葫芦画瓢，使用后就发现数据可以save/update了。虽然大体上知道注解是走的AOP那一套来实现的，但是为什么不加注解就无法操作数据库呢？所以这里就有必要来研究一下了。  

搞清楚底层之前，先想为啥会有这个注解存在：在JDBC的时候是可以显示用编程方式处理事务的，在JPA中也是可以的：

```java
UserTransaction utx = entityManager.getTransaction(); 
    try { 
        utx.begin(); 
        businessLogic();
        utx.commit(); 
    } catch(Exception ex) { 
        utx.rollback(); 
        throw ex; 
    }
```

但是，很明显，每个操作我都这样去搞的话，我们这些程序员可真就要累成狗了，所以Spring就给我们搞了一个东西：**@Transactional**：

```java
@Transactional
    public void businessLogic() {
        
    }
```

当然，用起来丝滑，但是如果不知道@Transactional注解里面干了些啥的话，这是很可怕的一个事情。
##### 大致原理
运行配置@Transactional注解的测试类的时候，具体会发生如下步骤：  
- 事务开始时，通过AOP机制，生成一个代理connection对象，并将其放入DataSource实例的某个与DataSourceTransactionManager相关的某处容器中。在接下来的整个事务中，客户代码都应该使用该connection连接数据库，执行所有数据库命令（不使用该connection连接数据库执行的数据库命令，在本事务回滚的时候得不到回滚）
- 事务结束时，回滚在上面骤中得到的代理connection对象上执行的数据库命令，然后关闭该代理connection对象

网上也有这样的介绍：
>spring定义了@Transactional注解，基于AbstractBeanFactoryPointcutAdvisor、StaticMethodMatcherPointcut、MethodInterceptor的aop编程模式，增强了添加@Transactional注解的方法。同时抽象了事务行为为PlatformTransactionManager(事务管理器)、TransactionStatus(事务状态)、TransactionDefinition(事务定义)等形态。最终将事务的开启、提交、回滚等逻辑嵌入到被增强的方法的前后，完成统一的事务模型管理。

##### 注解常见属性
transactionManager：事务管理器  
propagation：传播行为定义，枚举类型，是spring独有的事务行为设计，默认为PROPAGATION_REQUIRED（支持当前事务，不存在则新建）  
isolation：隔离级别，对应数据库的隔离级别实现，mysql默认的隔离级别是 read-committed  
timeout：超时时间，默认使用数据库的超时，mysql默认的事务等待超时为5分钟  
readOnly：是否只读，默认是false  
rollbackFor：异常回滚列表，默认的是RuntimeException异常回滚  

##### tx:annotation-driven
待补充

##### 总结
从上面的分析可以看到，Spring使用AOP实现事务的统一管理，为开发者提供了很大的便利。但是，有部分开发人员会误用这个便利，基本都是下面这两种情况：

1.A类的a1方法没有标注@Transactional，a2方法标注@Transactional，在a1里面调用a2；

2.将@Transactional注解标注在非public方法上。

第一种为什么是错误用法，原因很简单，a1方法是目标类A的原生方法，调用a1的时候即直接进入目标类A进行调用，在目标类A里面只有a2的原生方法，在a1里调用a2，即直接执行a2的原生方法，并不通过创建代理对象进行调用，所以并不会进入TransactionInterceptor的invoke方法，不会开启事务。

@Transactional的工作机制是基于AOP实现的，而AOP是使用动态代理实现的，动态代理要么是JDK方式、要么是Cglib方式。如果是JDK动态代理的方式，根据上面的分析可以知道，目标类的目标方法是在接口中定义的，也就是必须是public修饰的方法才可以被代理。如果是Cglib方式，代理类是目标类的子类，理论上可以代理public和protected方法，但是Spring在进行事务增强是否能够应用到当前目标类判断的时候，遍历的是目标类的public方法，所以Cglib方式也只对public方法有效。