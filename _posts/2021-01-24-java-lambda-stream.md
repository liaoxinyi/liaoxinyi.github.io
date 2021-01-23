---
layout:     post
title:      "拥抱Java8-01"
subtitle:   "Lambda"
date:       2021-01-24
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/design-handler-bg.jpg"
tags:
    - Java

---
> 资料来源于知乎上各位前辈（古时的风筝等）

### 前言
一直没能开始把平日里开发时使用的Java8的相关内容总结一下，今天正好抽空记录一下。话不多说，直接开搞。
### Lambda表达式
Java 8 是在 2014 年发布的，距离现在已经很久了，平日里使用的多得无非是ForEach等等，这里进行一些总结和归纳。先看看解释

>Lambda 表达式是一个匿名函数，Lambda表达式基于数学中的λ演算得名，直接对应于其中的lambda抽象，是一个匿名函数，即没有函数名的函数。Lambda表达式可以表示闭包。

```java
// 无参数，无返回值
() -> log.info("Lambda")
 // 有参数，有返回值
(int a, int b) -> { a+b }
//匿名内部类
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("快速新建并启动一个线程");
    }
}).run();
```

上面等价于

```java
log.info("Lambda");
private int plus(int a, int b){
    return a+b;
}
new Thread(()->{
    System.out.println("快速新建并启动一个线程");
}).run();
```

### 方法引用
什么事方法引用？看一下解释

>方法引用的出现，使得我们可以将一个方法赋给一个变量或者作为参数传递给另外一个方法。`::`双冒号作为方法引用的符号

比如：

```java
//引用 Integer类的 parseInt方法
Function<String, Integer> s = Integer::parseInt;
Integer i = s.apply("10");
//引用 Integer类的 compare方法
Comparator<Integer> comparator = Integer::compare;
int result = comparator.compare(100,10);
```

##### 什么样的方法可以被引用？
任何能够访问到的方法都可以被引用。
##### 返回值到底是什么类型？
- 返回的类型是 Java 8 专门定义的函数式接口，这类接口用 `@FunctionalInterface` 注解，一般常用`Function`  
- 引用方法的**参数个数**、**类型**，**返回值类型**要和函数式接口中的方法声明一一对应才行

比如 Integer.parseInt方法定义如下：

```java
public static int parseInt(String s) throws NumberFormatException {
    return parseInt(s,10);
}
```

首先parseInt方法的参数个数是 1 个，而返回的类型Function中的 apply方法参数个数也是 1 个，参数个数对应上了。同时，apply方法的参数类型和返回类型是泛型类型，所以肯定能和 parseInt方法对应上

##### 自己造一个函数式接口
上面对于方法引用可能还是不是很清楚，这里举一个例子来进行说明：

- 首先定义一个函数式接口

```java
@FunctionalInterface
public interface KiteFunction<T, R, S> {

    /**
     * 定义一个双参数的方法
     * @param t
     * @param s
     * @return
     */
    R run(T t,S s);
}
```

注意：函数式接口中只能声明一个可被实现的方法，你不能声明了一个 run方法，又声明一个 start方法，到时候编译器就不知道用哪个接收了。而用default 关键字修饰的方法则没有影响

- 定义一个实际会使用的方法

```java
public class FunctionTest {
    public static String DateFormat(LocalDateTime dateTime, String partten) {
        DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern(partten);
        return dateTime.format(dateTimeFormatter);
    }
}
```

- 用方法引用的方式调用

```java
//正常情况下的使用
FunctionTest.DateFormat();
//用函数式方式使用
KiteFunction<LocalDateTime,String,String> functionDateFormat = FunctionTest::DateFormat;
String dateString = functionDateFormat.run(LocalDateTime.now(),"yyyy-MM-dd HH:mm:ss");
```

### Stream API
Stream 是 Java 8 中集合数据处理的利器，很多本来复杂、需要写很多代码的方法，比如过滤、分组等操作，往往使用 Stream 就可以在一行代码搞定，当然也因为 Stream 都是链式操作，一行代码可能会调用好几个方法

Collection接口提供了 `stream()`方法，让我们可以在一个集合方便的使用 Stream API 来进行各种操作。值得注意的是，我们执行的任何操作都不会对源集合造成影响，你可以同时在一个集合上提取出多个 stream 进行操作

看 Stream 接口的定义，继承自 `BaseStream`，基本上所有的接口声明都是接收方法引用类型的参数，比如 `filter`方法，接收了一个 `Predicate`类型的参数，它就是一个函数式接口，常用来作为条件比较、筛选、过滤用，JPA中也使用了这个函数式接口用来做查询条件拼接

##### 