---
layout:     post
title:      "拥抱Java8-01"
subtitle:   "Lambda"
date:       2021-01-24
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java8-stream-bk.png"
tags:
    - Java

---
> 资料来源于知乎上各位前辈（古时的风筝、Java团长等）

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
可能经常会看到两个冒号的代码,`::`，这个其实就是方法引用。什么是方法引用？看一下解释

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

### Stream介绍
##### 是什么
Stream 是 Java 8 中集合数据处理的利器，很多本来复杂、需要写很多代码的方法，比如过滤、分组等操作，往往使用 Stream 就可以在一行代码搞定，当然也因为 Stream 都是链式操作，一行代码可能会调用好几个方法

看 Stream 接口的定义，继承自 `BaseStream`，基本上所有的接口声明都是接收方法引用类型的参数，比如 `filter`方法，接收了一个 `Predicate`类型的参数，它就是一个函数式接口，常用来作为条件比较、筛选、过滤用，JPA中也使用了这个函数式接口用来做查询条件拼接

Stream对流的操作分为两种

- 中间操作，每次返回一个新的流，可以有多个

- 终端操作，每个流只能进行一次终端操作，终端操作结束后流无法再次使用。终端操作会产生一个新的集合或值

##### 特点
- Stream不存储数据，而是按照特定的规则对数据进行计算，一般会输出结果

- Stream不会改变数据源，通常情况下会产生一个新的集合或一个值

- Stream具有延迟执行特性，只有调用终端操作时，中间操作才会执行

- Stream中的元素是以Optional类型存在的

##### Stream和ParallelStream
- stream是顺序流，由主线程按顺序对流执行操作，而parallelStream是并行流，内部以多线程并行执行的方式对流进行操作，但前提是流中的数据处理没有顺序要求

- 并行流除了直接创建外，可以通过由顺序流转换而来：`Optional<Integer> findFirst = list.stream().parallel().filter(x->x>6).findFirst();`

### Stream的创建
平时在开发的时候，Stream一般是通过集合数组创建的
##### Collection.stream()

```java
List<String> list = Arrays.asList("a", "b", "c");
// 创建一个顺序流
Stream<String> stream = list.stream();
// 创建一个并行流
Stream<String> parallelStream = list.parallelStream();
```

##### Arrays.stream(T[] array)

```java
int[] array={1,3,5,6,8};
IntStream stream = Arrays.stream(array);
```
##### Stream#of()/iterate()/generate()

```java
Stream<Integer> stream = Stream.of(1, 2, 3, 4, 5, 6);
Stream<Integer> stream2 = Stream.iterate(0, (x) -> x + 3).limit(4);
stream2.forEach(System.out::println); // 0 2 4 6 8 10
Stream<Double> stream3 = Stream.generate(Math::random).limit(3);
stream3.forEach(System.out::println);
```
### Stream的常规使用
##### foreach/findFirst/findAny/anyMatch

```java
List<Integer> list = Arrays.asList(7, 6, 9, 3, 8, 2, 1);
// 遍历输出符合条件的元素
list.stream().filter(x -> x > 6).forEach(System.out::println);
// 匹配第一个
Optional<Integer> findFirst = list.stream().filter(x -> x > 6).findFirst();
// 匹配任意（适用于并行流）
Optional<Integer> findAny = list.parallelStream().filter(x -> x > 6).findAny();
// 是否包含符合特定条件的元素
boolean anyMatch = list.stream().anyMatch(x -> x < 6);
System.out.println("匹配第一个值：" + findFirst.get());
System.out.println("匹配任意一个值：" + findAny.get());
System.out.println("是否存在大于6的值：" + anyMatch);
```

##### 过滤(filter)

- 过滤后执行操作

```java
List<Integer> list = Arrays.asList(6, 7, 3, 8, 1, 2, 9);
Stream<Integer> stream = list.stream().filter(x -> x > 7).forEach(System.out::println);
```

- 过滤后形成新的集合（依赖`collect`方法）

```java
List<String> fiterList = personList.stream().filter(x -> x.getSalary() > 8000).map(Person::getName).collect(Collectors.toList());
```

##### 聚合(max/min/count)

- 获取String集合中最长的元素

```java
List<String> list = Arrays.asList("adnm", "admmt", "pot", "xbangd", "weoujgsd");
Optional<String> max = list.stream().max(Comparator.comparing(String::length));
System.out.println("最长的字符串：" + max.get());
```

- 获取Integer集合中的最大值

```java
List<Integer> list = Arrays.asList(7, 6, 9, 4, 11, 6);
// 自然排序
Optional<Integer> max = list.stream().max(Integer::compareTo);
// 自定义排序
Optional<Integer> max2 = list.stream().max(new Comparator<Integer>() {
	@Override
	public int compare(Integer o1, Integer o2) {
        // use ur own method
		return o1.compareTo(o2);
	}
});
System.out.println("自然排序的最大值：" + max.get());
		System.out.println("自定义排序的最大值：" + max2.get());
```

- 获取对象集合中的最大值

```java
List<Person> personList = new ArrayList<Person>();
personList.add(new Person("Tom", 8900, 23, "male", "New York"));
personList.add(new Person("Jack", 7000, 25, "male", "Washington"));
personList.add(new Person("Lily", 7800, 21, "female", "Washington"));
personList.add(new Person("Anni", 8200, 24, "female", "New York"));
personList.add(new Person("Owen", 9500, 25, "male", "New York"));
personList.add(new Person("Alisa", 7900, 26, "female", "New York"));

Optional<Person> max = personList.stream().max(Comparator.comparingInt(Person::getSalary));
System.out.println("员工工资最大值：" + max.get().getSalary());
```

- 计算Integer集合中大于6的元素的个数

```java
List<Integer> list = Arrays.asList(7, 6, 4, 8, 2, 11, 9);
long count = list.stream().filter(x -> x > 6).count();
System.out.println("list中大于6的元素个数：" + count);
```

##### 映射(map/flatMap)
可以将一个流的元素按照一定的映射规则映射到另一个流中。分为`map`和`flatMap`

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-Lambda-02.jpg)

map：接收一个函数作为参数，该函数会被应用到每个元素上，并将其映射成一个新的元素

flatMap：接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流

- 英文字符串数组的元素全部改为大写。整数数组每个元素+3

```java
String[] strArr = { "abcd", "bcdd", "defde", "fTr" };
List<String> strList = Arrays.stream(strArr).map(String::toUpperCase).collect(Collectors.toList());
List<Integer> intList = Arrays.asList(1, 3, 5, 7, 9, 11);
List<Integer> intListNew = intList.stream().map(x -> x + 3).collect(Collectors.toList());
System.out.println("每个元素大写：" + strList);
System.out.println("每个元素+3：" + intListNew);
```

- 对象集合的映射

```java
// 不改变原来员工集合的方式
List<Person> personListNew = personList.stream().map(person -> {
	Person personNew = new Person(person.getName(), 0, 0, null, null);
	personNew.setSalary(person.getSalary() + 10000);
	return personNew;
}).collect(Collectors.toList());
// 改变原来员工集合的方式
List<Person> personListNew2 = personList.stream().map(person -> {
	person.setSalary(person.getSalary() + 10000);
	return person;
}).collect(Collectors.toList());
//常用减少内存的操作
int asInt = javaProgrammers.stream()
                                    .mapToInt(Person::getSalary)//返回数值流，减少拆箱封箱操作，避免占用内存  IntStream
                                    .reduce((x, y) -> x += y)// int
                                    .getAsInt(); //return int
```
- 字符串拆分然后合并成一个新的字符数组

```java
List<String> list = Arrays.asList("m,k,l,a", "1,3,5,7");
List<String> listNew = list.stream().flatMap(s -> {
	// 将每个元素转换成一个stream
	String[] split = s.split(",");
	Stream<String> s2 = Arrays.stream(split);
	return s2;
}).collect(Collectors.toList());
```

##### 归约(reduce)
是把一个流缩减成一个值，能实现对集合求和、求乘积和求最值操作

- `Optional<T> reduce(BinaryOperator<T> accumulator);`

参数列表为一个函数接口`BinaryOperator<T>`，说明该方法需要一个函数式接口参数，该函数式接口需要两个参数，返回一个结果(reduce中返回的结果会作为下次累加器计算的第一个参数)，也就是累加器,最终得到一个Optional对象

```java
/*解析：
     1. reduce(BinaryOperator<T> accumulator)    reduce方法接受一个函数，这个函数有两个参数
     2. 第一个参数是上次函数执行的返回值（也称为中间结果），第二个参数是stream中的元素，这个函数把这两个值相加，得到的和会被赋值给下次执行这个函数的第一个参数
 *注意：
     1.第一次执行的时候第一个参数的值是Stream的第一个元素，第二个参数是Stream的第二个元素
     2.方法返回值类型是Optional
 */
int asInt = javaProgrammers.stream()
                                    .mapToInt(Person::getSalary)//返回数值流，减少拆箱封箱操作，避免占用内存  IntStream
                                    .reduce((x, y) -> x += y)// int
                                    .getAsInt(); //return int
```

- `T reduce(T identity, BinaryOperator<T> accumulator);`

与第一种变形相同的是都会接受一个BinaryOperator函数接口，不同的是其会接受一个identity参数，identity参数与Stream中数据同类型，相当于一个的初始值，通过累加器accumulator迭代计算Stream中的数据，得到一个跟Stream中数据相同类型的最终结果

```java
/*注意：
 *      1.与方式一相比设置了累加器的初始值，参数一（x）则不再是Stream中的第一个数据而是设置的初始值（10000）其他相同
 */
int reduce = javaProgrammers.stream().mapToInt(Person::getSalary).reduce(10000, (x, y) -> x += y);
```

- `U reduce(U identity,BiFunction<U, ? super T, U> accumulator,BinaryOperator<U> combiner);`

参数列表为一个函数接口`BinaryOperator<T>`，说明该方法需要一个函数式接口参数，该函数式接口需要两个参数，返回一个结果(reduce中返回的结果会作为下次累加器计算的第一个参数)，也就是累加器,最终得到一个Optional对象

```java
/*解析：
     1. 第一个参数：返回实例u，传递你要返回的U类型对象的初始化实例u
     2. 第二个参数：累加器accumulator，可以使用lambda表达式，声明你在u上累加你的数据来源t的逻辑，例如(u,t)->u.sum(t),此时lambda表达式的行参列表是返回实例u和遍历的集合元素t，函数体是在u上累加t
     3. 第三个参数：参数组合器combiner，接受lambda表达式
 *注意：
     第三个参数用来处理并发操作，如何处理数据的重复性，应多做考虑，否则会出现重复数据
 */
//简化版代码
ArrayList<Integer> newList = new ArrayList<>();

        ArrayList<Integer> accResult_s = Stream.of(1,2,3,4)
                .reduce(newList,
                        (acc, item) -> {
                            acc.add(item);
                            return acc;
                        }, (acc, item) -> null);
                        
//原理版代码
ArrayList<Integer> accResult_ = Stream.of(1, 2, 3, 4)
               //第一个参数，初始值为ArrayList
               .reduce(new ArrayList<Integer>(),
                       //第二个参数，实现了BiFunction函数式接口中apply方法，并且打印BiFunction
                       new BiFunction<ArrayList<Integer>, Integer, ArrayList<Integer>>() {
                           @Override
                           public ArrayList<Integer> apply(ArrayList<Integer> acc, Integer item) {

                               acc.add(item);
                               System.out.println("item: " + item);
                               System.out.println("acc+ : " + acc);
                               System.out.println("BiFunction");
                               return acc;
                           }
                           //第三个参数---参数的数据类型必须为返回数据类型，改参数主要用于合并多个线程的result值
                           // （Stream是支持并发操作的，为了避免竞争，对于reduce线程都会有独立的result）
                       }, new BinaryOperator<ArrayList<Integer>>() {
                           @Override
                           public ArrayList<Integer> apply(ArrayList<Integer> acc, ArrayList<Integer> item) {
                               System.out.println("BinaryOperator");
                               acc.addAll(item);
                               System.out.println("item: " + item);
                               System.out.println("acc+ : " + acc);
                               System.out.println("--------");
                               return acc;
                           }
                       });
```

- reduce也可以用来求最大值

```java
// 求最高工资方式1：
Integer maxSalary = personList.stream().reduce(0, (max, p) -> max > p.getSalary() ? max : p.getSalary(),Integer::max);
// 求最高工资方式2：
Integer maxSalary2 = personList.stream().reduce(0, (max, p) -> max > p.getSalary() ? max : p.getSalary(),(max1, max2) -> max1 > max2 ? max1 : max2);
```

##### 排序(sorted)

- sorted()：自然排序，流中元素需实现Comparable接口

- sorted(Comparator com)：Comparator排序器自定义排序

```java
// 按工资升序排序（自然排序）
List<String> newList = personList.stream().sorted(Comparator.comparing(Person::getSalary)).map(Person::getName).collect(Collectors.toList());
// 按工资倒序排序
List<String> newList2 = personList.stream().sorted(Comparator.comparing(Person::getSalary).reversed()).map(Person::getName).collect(Collectors.toList());
// 先按工资再按年龄升序排序
List<String> newList3 = personList.stream().sorted(Comparator.comparing(Person::getSalary).thenComparing(Person::getAge)).map(Person::getName).collect(Collectors.toList());
// 先按工资再按年龄自定义排序（降序）
List<String> newList4 = personList.stream().sorted((p1, p2) -> {
	if (p1.getSalary() == p2.getSalary()) {
		return p2.getAge() - p1.getAge();
	} else {
		return p2.getSalary() - p1.getSalary();
	}
}).map(Person::getName).collect(Collectors.toList());
```

##### 合并/去重/限制/跳过(concat/distinct/limit/skip)
流也可以进行合并、去重、限制、跳过等操作

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-Lambda-03.jpg)

```java
String[] arr1 = { "a", "b", "c", "d" };
String[] arr2 = { "d", "e", "f", "g" };
Stream<String> stream1 = Stream.of(arr1);
Stream<String> stream2 = Stream.of(arr2);
// concat:合并两个流 distinct：去重
List<String> newList = Stream.concat(stream1, stream2).distinct().collect(Collectors.toList());
// limit：限制从流中获得前n个数据
List<Integer> collect = Stream.iterate(1, x -> x + 2).limit(10).collect(Collectors.toList());
// skip：跳过前n个数据
List<Integer> collect2 = Stream.iterate(1, x -> x + 2).skip(1).limit(5).collect(Collectors.toList());
```

### Stream的collect使用
collect把一个流收集起来，最终可以是收集成一个值也可以收集成一个新的集合，主要依靠`java.util.stream.Collectors`类内置的静态方法
##### Collectors归集(toList/toSet/toMap/toCollection/toConcurrentMap)

```java
List<Person> personList = new ArrayList<Person>();
personList.add(new Person("Tom", 8900, 23, "male", "New York"));
personList.add(new Person("Jack", 7000, 25, "male", "Washington"));
personList.add(new Person("Lily", 7800, 21, "female", "Washington"));
personList.add(new Person("Anni", 8200, 24, "female", "New York"));

Map<?, Person> map = personList.stream().filter(p -> p.getSalary() > 8000).collect(Collectors.toMap(Person::getName, p -> p));
```

##### Collectors统计(averagingInt等)
- 计数：count平均值：averagingInt、averagingLong、averagingDouble

- 最值：maxBy、minBy

- 求和：summingInt、summingLong、summingDouble

- 统计以上所有：summarizingInt、summarizingLong、summarizingDouble

```java
// 求总数
Long count = personList.stream().collect(Collectors.counting());
// 求平均工资
Double average = personList.stream().collect(Collectors.averagingDouble(Person::getSalary));
// 求最高工资
Optional<Integer> max = personList.stream().map(Person::getSalary).collect(Collectors.maxBy(Integer::compare));
// 求工资之和
Integer sum = personList.stream().collect(Collectors.summingInt(Person::getSalary));
// 一次性统计所有信息
DoubleSummaryStatistics collect = personList.stream().collect(Collectors.summarizingDouble(Person::getSalary));
```

##### Collectors分组(partitioningBy/groupingBy)

- 分区：将stream按条件分为两个Map，比如员工按薪资是否高于8000分为两部分

- 分组：将集合分为多个Map，比如员工按性别分组。有单级分组和多级分组

![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-Lambda-02.jpg)  

```java
// 将员工按薪资是否高于8000分组
Map<Boolean, List<Person>> part = personList.stream().collect(Collectors.partitioningBy(x -> x.getSalary() > 8000));
// 将员工按性别分组
Map<String, List<Person>> group = personList.stream().collect(Collectors.groupingBy(Person::getSex));
// 将员工先按性别分组，再按地区分组
Map<String, Map<String, List<Person>>> group2 = personList.stream().collect(Collectors.groupingBy(Person::getSex, Collectors.groupingBy(Person::getArea)));
```

##### Collectors接合(joining)
`joining`可以将stream中的元素用特定的连接符（没有的话，则直接连接）连接成一个字符串

```java
String names = personList.stream().map(p -> p.getName()).collect(Collectors.joining(","));
List<String> list = Arrays.asList("A", "B", "C");
String string = list.stream().collect(Collectors.joining("-"));
```

##### Collectors归约(reducing)
Collectors类提供的reducing方法，相比于stream本身的reduce方法，增加了对自定义归约的支持

```java
// 每个员工减去起征点后的薪资之和（这个例子并不严谨，但一时没想到好的例子）
Integer sum = personList.stream().collect(Collectors.reducing(0, Person::getSalary, (i, j) -> (i + j - 5000)));
System.out.println("员工扣税薪资总和：" + sum);
// stream的reduce
Optional<Integer> sum2 = personList.stream().map(Person::getSalary).reduce(Integer::sum);
System.out.println("员工薪资总和：" + sum2.get());
```