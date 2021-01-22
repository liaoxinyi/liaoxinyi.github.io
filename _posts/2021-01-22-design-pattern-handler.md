---
layout:     post
title:      "设计模式01-责任链"
subtitle:   "责任链，待补充行为类模式是什么意思"
date:       2021-01-22
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/design-handler-bg.jpg"
tags:
    - 设计模式

---
> 资料来源于知乎上各位前辈（小灰、林东洲等）

### 前言
前段时间在产品中设计过一个流程，大体上是需要针对同一份来自中间件的消息报文**依次**进行一系列的处理流程。比如报文中的某些字段是否存在，存在的字段是否满足特定的要求等等。最开始开发完成后，对于这条消息报文的处理逻辑比较冗长，想要优化缺发现没有办法提取出方法来。这时候碰巧遇到了一篇介绍责任链设计模式的文章，发现责任链模式特别适合这样的处理场景，相比现在荣昌的处理流程，责任链模式的代码更加顺畅、清晰，而且最主要的是后期增加和调整处理顺序十分方便。话不多说，直接开搞。
### 场景描述
> 场景分享来源于知乎前辈林东洲

- 场景描述

你要去给某公司借款 1 万元，当你来到柜台的时候向柜员发起 "借款 1 万元" 的请求时，柜员认为金额太多，处理不了这样的请求，他转交这个请求给他的组长，组长也处理不了这样的请求，那么他接着向经理转交这样的请求。

- 代码示意

```java
public void test(Request request) {
    int money = request.getRequestMoney();
    if(money <= 1000) {
        Clerk.response(request);	
    } else if(money <= 5000) {
        Leader.response(request);
    } else if(money <= 10000) {
        Manager.response(request);
    }
}
```

- 潜在问题

1. 代码臃肿: 实际应用中的判定条件通常不是这么简单地判断金额，也许需要复杂的操作，也许需要查询数据库等等，这就会产生许多额外的代码，如果判断条件再比较多的话，那么代码就会大量地堆积在同一个文件中

2. 耦合度高：如果我们想继续添加处理请求的类，那么就需要添加 else if 的判定条件；另外，这个条件判定的顺序也是写死的。如果想改变顺序，那么也只能修改这个条件语句

### 基本原理
责任链模式(Chain of Responsibility)使多个对象都有机会处理请求，从而避免请求的发送者和接受者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递该请求，直到有对象能够处理它

责任链模式其实属于行为类模式，至于什么是行为类模式（**暂时还不是很了解，等待后续来补充**）

### 几个角色
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/design-handler01.jpg)  

##### 抽象处理类
主要包含一个指向下一处理类的成员变量`nextHandler`和一个处理请求的方法handRequest，handRequest方法的主要思想是，如果满足处理的条件，则有本处理类来进行处理，否则由nextHandler来处理

```java
public abstract class XxxBaseHandler {

    protected XxxBaseHandler nextHandler = null;

    public void setNextHandler(XxxBaseHandler handler) {
        this.nextHandler = handler;
    }

    /**
     * 处理请求
     * @param Object
     */
    public abstract void handleObject(Object object);
}
```

##### 具体处理类
具体处理类的主要是对具体的处理逻辑和处理的适用条件进行实现

```java
@Component
public class XxxFirstHandler extends XxxBaseHandler {

    @Override
    public  void handleObject(Object object) {
        boolean continueHandle=Boolean.TRUE;
        //do something to change {@code continueHandle}  to {@code Boolean.FALSE}  and finish this handle.
        if (continueHandle&&null!=this.nextHandler) {
            this.nextHandler.handleObject(object);
        }
    }
}

@Component
public class XxxSecondHandler extends XxxBaseHandler {

    @Override
    public  void handleObject(Object object) {
        boolean continueHandle=Boolean.TRUE;
        //do something to change {@code continueHandle}  to {@code Boolean.FALSE}  and finish this handle.
        if (continueHandle&&null!=this.nextHandler) {
            this.nextHandler.handleObject(object);
        }
    }
}

@Component
public class XxxThirdHandler extends XxxBaseHandler {

    @Override
    public  void handleObject(Object object) {
        boolean continueHandle=Boolean.TRUE;
        //do something to change {@code continueHandle}  to {@code Boolean.FALSE}  and finish this handle.
        if (continueHandle&&null!=this.nextHandler) {
            this.nextHandler.handleObject(object);
        }
    }
}
```

##### 具体调用

```java
@Autowired
private XxxFirstHandler xxxFirstHandler;
@Autowired
private XxxSecondHandler xxxSecondHandler;
@Autowired
private XxxThirdHandler xxxThirdHandler;

@PostConstruct
private void initXxxHandler(){
    // 初始化，需要在@PostConstruct里面进行责任链的顺序确定，否则无法使用
    xxxFirstHandler.setNextHandler(xxxSecondHandler);
    xxxSecondHandler.setNextHandler(xxxThirdHandler);
}

//正式使用
xxxFirstHandler.handleObject(new Object());

```

### 实战

- 借款请求

```java
class BorrowRequest {
    private int requestMoney;
    public BorrowRequest(int money) {
        System.out.println("有新请求，需要借款 " + money + " 元");
        requestMoney = money;
    }
    public int getMoney() {
        return requestMoney;
    }
}
```

- 抽象职员类，用于实现责任链

```java
abstract class AbstractClerk {
    private AbstractClerk superior = null;
    protected String type;
    public void setSuperior(AbstractClerk superior) {
        this.superior = superior;
    } 
    public void approveRequest(BorrowRequest request) {
        if(getLimit() >= request.getMoney()) {
            System.out.println(getType() + "同意借款请求");
        }else {
            if(this.superior != null) {
                this.superior.approveRequest(request);
            }else {
                System.out.println("没有人能够同意借款请求");
            }
        }
    }
    public abstract int getLimit();
    public String getType() {
        return type;
    }
}
```

- 具体的员工类，并设置其额度

```java
class Clerk extends AbstractClerk{
    public Clerk() {
        super.type = "职员";
    }
    public int getLimit() {
        return 5000;
    }
}

class Leader extends AbstractClerk{
    public Leader() {
        super.type = "组长";
    }
    public int getLimit() {
        return 20000;
    }
}

class Manager extends AbstractClerk{
    public Manager() {
        super.type = "经理";
    }
    public int getLimit() {
        return 100000;
    }
}
```

- 正式使用

```java
public class Client {
    public static void main(String[] args) {
        AbstractClerk clerk = new Clerk();
        AbstractClerk leader = new Leader();
        AbstractClerk manager = new Manager();

        clerk.setSuperior(leader);
        leader.setSuperior(manager);

        //有人借款 10000 元
        clerk.approveRequest(new BorrowRequest(10000));

        //有人借款 111000 元
        clerk.approveRequest(new BorrowRequest(111000));

    }
}
```

### 总结
- 责任链模式与 if...else 相比，他的耦合性要低一些，因为它将条件判定分散到各个处理类中，并且这些处理类的优先处理顺序可以随意的设定，并且如果想要添加新的 handler 类也是十分简单的，这符合开放闭合原则

- 责任链模式带来了灵活性，但是在设置处理类前后关系时，一定要避免在链中出现循环引用的问题