---
layout:     post
title:      "春天一样的代码长啥样？Spring-01"
subtitle:   "Transactional注解到底干了啥？Spring的事务传播属性、为什么有的事务不生效？"
update-date:  2021-04-07
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-spring-bk-01.jpg"
tags:
    - Spring
    - Java
    - Transaction

---
> 资料来源于网络上各位前辈（wind瑞_等）

### 前言
有时候真想学会NARUTO的分身术，因为总感觉时间不够用。最近开始将之前在项目中研究的一些东西沉淀时，发现时间完全不够用。前面才开了一个Redis的坑，这里又开了Spring的坑，也不知道啥时候能够填的起来，但无论怎样，还是话不多说，直接开搞。
说道这个注解，都知道是用了AOP的思想，大致的原理如下图：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-spring-01-02.jpg)  
<center>Transactional注解的大致原理</center>  
**总的来说，就是依据AOP的思路，在业务代码的前后增加了切面，切面内容就是开启和关闭事务**
### AOP和Spring AOP
AOP是一种通用的编程思想，Java里有2种实现方式：  
- 方式一：Spring AOP  
基于动态代理实现：JDK代理或者Cglib代理  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-spring-01-01.jpg)  
<center>springAOP中的JDK代理和Cglib代理区别</center>  
注意：CGLib通过ASM动态操作指令生成了被代理类的子类，重写了目标类中的方法，但不能重写`private、final、static`方法  
- 方式二：AspectJ  
基于编译期实现

### Spring事务实现方式
Spring提供了很好事务管理机制，主要分为编程式事务和声明式事务两种：  
##### 编程式事务
是指在代码中手动的管理事务的提交、回滚等操作，代码侵入性比较强  
##### 声明式事务
基于AOP面向切面的，它将具体业务与事务处理部分解耦，代码侵入性很低，所以在实际开发中声明式事务用的比较多。声明式事务也有两种实现方式：  
- 基于TX和AOP的xml配置文件方式  
- 基于`@Transactional`注解，其大致描述为这样的过程：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-spring-01-03.jpg)  
<center>Transactional注解的示意</center>  

### @Transactional可以用在哪?
`@Transactional`可以作用在接口、类、类方法  
##### 作用于类
当把`@Transactional` 注解放在类上时，表示所有该类的public方法都配置相同的事务属性信息
### 作用于方法
当类配置了`@Transactional`，方法也配置了`@Transactional`，方法的事务会覆盖类的事务配置信息
### 作用于接口
不推荐这种使用方法，因为一旦标注在Interface上并且配置了Spring AOP 使用CGLib动态代理，将会导致`@Transactional`注解失效  

### @Transactional增强流程
最开始使用`@Transactional`注解的时候，是因为在SpringBoot中使用了JPA，发现查询的时候正常但是保存或者修改东西的时候，无法生效。百度一波后，结果告诉我要在方法上添加@Transactional注解，然后就依葫芦画瓢，使用后就发现数据可以save/update了。虽然大体上知道注解是走的AOP那一套来实现的，但是为什么不加注解就无法操作数据库呢？所以这里就有必要来研究一下了，看一下整个事务增强的大致过程：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-spring-01-04.jpg)  
<center>事务的增强执行过程，以cglib代理方式为例</center>  
如上图所示 `TransactionInterceptor` （事务拦截器）在目标方法执行前后进行拦截，大致流程为：  
- 获取事务的属性（`@Transactional`注解中的配置）  
- 加载配置中的TransactionManager  
- 获取TransactionInfo，通过`createTransactionIfNecessary`方法：主要是执行了JDBC的`con.setAutoCommit(false)`方法。同时处理了很多和数据库连接相关的`ThreadLocal`变量  
- 执行目标方法  
- 出现异常，尝试处理  
- 清理事务相关信息  
- 提交事务

```java
//1. 获取@Transactional注解的相关参数
TransactionAttributeSource tas = getTransactionAttributeSource();
// 2. 获取TransactionManager
final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
final PlatformTransactionManager tm = determineTransactionManager(txAttr);
final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);
// 标准声明式事务：如果事务属性为空 或者 非回调偏向的事务管理器
if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
	// Standard transaction demarcation with getTransaction and commit/rollback calls.
	// 3. 获取TransactionInfo，包含了tm和TransactionStatus，这里创建并开启事务
	TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
	Object retVal = null;
	try {
		// This is an around advice: Invoke the next interceptor in the chain.
		// This will normally result in a target object being invoked.
		// 4.执行目标方法
		retVal = invocation.proceedWithInvocation();
	}
	catch (Throwable ex) {
	   //5.如果有异常则回滚
		// target invocation exception
		completeTransactionAfterThrowing(txInfo, ex);
		throw ex;
	}
	finally {
	  // 6. 清理当前线程的事务相关信息
		cleanupTransactionInfo(txInfo);
	}
	// 7. 提交事务
	commitTransactionAfterReturning(txInfo);
	return retVal;
}

```

### 三大接口PlatformTransactionManager、TransactionDefinition、TransactionStatus
- `PlatformTransactionManager`是事务管理器  
- `TransactionDefinition`接口定义事务的超时时间、传播属性，隔离级别等  
- `TransactionStatus`为事务状态信息，如是否为新事务，是否完成等  
- 三大接口的关系  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-spring-01-05.jpg)  
<center>三大接口关系</center> 

### Stp1：获取@Transactional属性
- transactionManager：事务管理器  
- propagation：传播行为定义，枚举类型，是spring独有的事务行为设计，默认为PROPAGATION_REQUIRED（支持当前事务，不存在则新建）  
- isolation：隔离级别，对应数据库的隔离级别实现，mysql默认的隔离级别是 read-committed  
- timeout：超时时间，默认使用数据库的超时，mysql默认的事务等待超时为5分钟  
- readOnly：是否只读，默认是false  
- rollbackFor：异常回滚列表，默认的是RuntimeException异常（或继承自 RuntimeException 的异常）回滚，Error也会回滚，其他异常则需要手动指定  
- noRollbackFor：抛出指定的异常类型，不回滚事务，也可以指定多个异常类型

### Stp2：获取TransactionManager
##### TransactionManager接口 
`TransactionManager`接口是做什么的？**源代码里面竟然是一个空接口？**它保存着当前的数据源连接，对外提供对该数据源的事务提交回滚操作接口，同时实现了事务相关操作的方法。**一个数据源`DataSource`需要一个事务管理器**  
##### PlatformTransactionManager接口
继承自`TransactionManager`接口

```java
public interface PlatformTransactionManager {	
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
 
	void commit(TransactionStatus status) throws TransactionException;
 	
	void rollback(TransactionStatus status) throws TransactionException;
}
```

##### TransactionDefinition接口
定义事务的超时时间、传播属性、隔离级别等（通过接口的成员变量来定义的），并提供了获取默认事务传播属性以及隔离级别的方法，可以理解为这是事务的常量定义类  
该接口的实现`DefaultTransactionDefinition`  

```java
public class DefaultTransactionDefinition implements TransactionDefinition, Serializable {
    private int propagationBehavior = PROPAGATION_REQUIRED;
    private int isolationLevel = ISOLATION_DEFAULT;
    private int timeout = TIMEOUT_DEFAULT;
    private boolean readOnly = false;
    //略
}
```

- 事务的传播属性

|对事务的要求|传播属性|具体含义|  
|-|-|-|  
|必须要有事务|__PROPAGATION_MANDATORY__|如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常|  
|必须要有事务|__PROPAGATION_REQUIRES_NEW__|每次都创建一个新的事务，如果当前存在事务，则把当前事务挂起再创建一个新事务|  
|必须要有事务|__PROPAGATION_NESTED__|如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则创建一个新的事务|  
|必须要有事务|__PROPAGATION_REQUIRED__|如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务|  
|事务可有可无|__PROPAGATION_SUPPORTS__|如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行|  
|不能有事务的|__PROPAGATION_NEVER__|如果当前存在事务，则抛出异常；如果当前没有事务，则以非事务的方式继续运行|  
|不能有事务的|__PROPAGATION_NOT_SUPPORTED__|如果当前存在事务，则把当前事务挂起；如果当前没有事务，则以非事务的方式继续运行|  

- 事务挂起是什么？  
    挂起事务，指的是将当前事务的属性如事务名称，隔离级别等属性保存在一个变量中，同时将当前线程中所有和事务相关的ThreadLocal变量设置为从未开启过线程一样。Spring维护着一个当前线程的事务状态，用来判断当前线程是否在一个事务中以及在一个什么样的事务中，**挂起事务后，当前线程的事务状态就好像没有事务**  
- 事务的隔离级别采用底层数据库默认的隔离级别  
- 超时时间采用底层数据库默认的超时时间  
- 是否只读为`false`

### Stp3：获取TransactionStatus
##### TransactionStatus接口 
TransactionStatus本身更多存储的是事务的一些状态信息：**比如当前正在运行的是哪个事务**  
```java
package org.springframework.transaction;
import java.io.Flushable;

public interface TransactionStatus extends TransactionExecution, SavepointManager, Flushable {
    boolean hasSavepoint();
    void flush();
}
```

### 为什么有的事务不生效？
- 数据库引擎是否支持存储引擎，比如MyISAM引擎就不支持事务  
- 没有指定`rollbackFor`参数  
- 没有指定合适的`transactionManager`参数，默认的transactionManager并不是支持在一个事务中操作多个类型的数据库  
- 如果AOP使用了JDK动态代理（**Spring默认的代理方式**），对象内部方法互相调用时是不会触发切面增强的，也就是说此时`@Transactional注解`无效  
    很简单，因为切面是在目标方法之外的，目标方法的内部是通过调用原始方法来执行的，所以此时不是代理对象的方法，自然切面就不会生效了。也就是说：**只有当事务方法被当前类以外的代码调用时，才会由Spring生成的代理对象来管理**  
    可以通过自己注入自己的方式来调用或者把方法抽取到另外一个service，或者`将该Service自己注入自己，然后记得配置@Lazy属性，最后通过该注入的bean来调用该方法即可`  
- 如果AOP使用了CGLIB代理且事务方法未被`public`修饰，因为在AOP的增强逻辑里面，不是`public`则不会获取`@Transactional`的属性配置信息。所以，`protected、private` 修饰的方法上使用 `@Transactional` 注解时，虽然不会有任何报错，但是事务依旧无效  
- 异常被`catch`“吃了”导致`@Transactional`失效  

```java
@Transactional
private Integer A() throws Exception {
    int insert = 0;
    try {
        CityInfoDict cityInfoDict = new CityInfoDict();
        cityInfoDict.setCityName("2");
        cityInfoDict.setParentCityId(2);
        /**
         * A 插入字段为 2的数据
         */
        insert = cityInfoDictMapper.insert(cityInfoDict);
        /**
         * B 插入字段为 3的数据
         */
        b.insertB();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

**如果B方法内部抛了异常，而A方法此时try catch了B方法的异常，那这个事务不能正常回滚，且会抛出如下异常：**  

`org.springframework.transaction.UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only`

因为当ServiceB中抛出了一个异常以后，ServiceB标识当前事务需要rollback。但是ServiceA中由于手动捕获这个异常并进行处理，ServiceA认为当前事务应该正常commit。此时就出现了前后不一致，也就是因为这样，抛出`UnexpectedRollbackException`异常

Spring的事务是在调用业务方法之前开始的，业务方法执行完毕之后才执行提交或者回滚，事务是否执行取决于是否抛出runtime异常。如果抛出runtime exception 并在业务方法中没有catch到的话，事务会回滚。