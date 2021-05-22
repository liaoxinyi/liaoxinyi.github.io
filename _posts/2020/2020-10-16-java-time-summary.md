---
layout:     post
title:      "时间日期那些事"
subtitle:   "Some java API about time."
update-date:       2020-10-16
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/time.jpg"
tags:
    - Java
---
> Calendar、Date、Time、TimeStamp、LocalTime、LocalDate、LocalDateTime、Instant、Duration、Period...。

### Java8之前
##### java.util.Date
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/20201012084926.png)
<center>Date、Time、Timstamp的UML类图</center>
- Date类表示的是特定的，瞬间的，它能精确毫秒
- 由它衍生的子类：java.sql.Date、java.sql.Time和java.sql.Timestamp
```java
//初始化
// 默认创建一个本地时间, long类型
// 从1970-1-1 0:0:0开始计算的毫秒数
Date date = new Date();
Date date1 = new Date(1000L);
Date date = new Date(System.currentTimeMillis());
// 打印出北京时间 Thu Jan 01 08:00:01 CST 1970
System.out.println(date1);
// 打印出格林标准时间 1 Jan 1970 00:00:01 GMT
System.out.println(date1.toGMTString());
// 返回与格林时间的时差, 以分钟计时, 正好是8个小时, 此函数输出-480   则北京时间-480分钟等于格林时间
date1.getTimezoneOffset();
long m = date1.getTime();
// 打印出date到1970年1月1日的毫秒数
System.out.println("m = " + m);
// 比较时间
// 返回boolean类型
date.after(date1);
date.before(date1);
// 返回-1 1 0
date.compareTo(date1);
```

##### java.util.Calendar
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-time-calendar.png)
<center>Calendar的UML类图</center>
- 从JDK1.1开始，在处理日期和时间时，系统推荐使用Calendar类进行实现
- Calendar它是一种抽象类，相比Date它在操作日历的时候提供了一些方法来操作日历字段
- 一般认为Calendar是Date的加强版，实际使用的频率要比Date更多
```java
Calendar calendar = new GregorianCalendar();
// 设置日历时间
calendar.set(Calendar.YEAR, 2019);
calendar.set(Calendar.MONTH, 5);
calendar.set(Calendar.DAY_OF_MONTH, 26);
//使用Date类设置calendar时间
calendar.setTime(new Date());
//取得日历时间 calendar.getTime();  返回一个Date对象
// 输出Wed Jun 26 12:58:42 CST 2019
System.out.println(calendar.getTime().toString());
//使用日历取得时间偏移
// 输出Tue Jun 26 12:58:42 CST 2029
calendar.add(Calendar.YEAR, 10);
System.out.println(calendar.getTime().toString());
```

##### 相互转换
- Calendar 转化 String
```java
Calendar calendat = Calendar.getInstance();
SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
String dateStr = sdf.format(calendar.getTime());
```
- String 转化Calendar
```java
String str="2012-5-27";
SimpleDateFormat sdf= new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
Date date =sdf.parse(str);
Calendar calendar = Calendar.getInstance();
calendar.setTime(date);
```
- TimeStamp转化String
```java
SimpleDateFormat sdf= new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
TimeStamp ts =...;
String result=sdf.format(ts);
```

##### 老版本总结
- 线程安全: <font color=red>Date和Calendar不是线程安全的</font>，需要编写额外的代码处理线程安全问题
- API设计和易用性: 涉及到一些日期或者时间字符的处理问题时，不够灵活
- 时间的格式化的时候一般采用SimpleDateFormat，但SimpleDateFormat是线程不安全的  
原因一：因为SimpleDateFormat#format方法中calendar是共享变量，并且这个共享变量没有做线程安全控制。当多个线程同时使用相同的SimpleDateFormat对象【如用static修饰的SimpleDateFormat】调用format方法时，多个线程会同时调用calendar.setTime方法，可能一个线程刚设置好time值另外的一个线程马上把设置的time值给修改了导致返回的格式化时间可能是错误的  
原因二：SimpleDateFormat#parse方法也是线程不安全的  
所以，避免线程之间共享一个SimpleDateFormat对象，每个线程使用时都创建一次SimpleDateFormat对象。加上Date对于时间的处理真的很不方便，而且获取年月日的大多方法都是已经被废弃了，千言万语，还是尝鲜Java8吧

### Java8的新特性：java.time
**<font color=red>新增加的日期类都是线程安全的</font>**  
**Instant**：时间戳  
**Duration**：持续时间，时间差  
**LocalDate**：只包含日期，比如：2019-02-02  
**LocalTime**：只包含时间，比如：23:12:10  
**LocalDateTime**：包含日期和时间，比如：2019-02-02 23:14:21  
**Period**：时间段  
**ZoneOffset**：时区偏移量，比如：+8:00  
**ZonedDateTime**：带时区的时间  
**Clock**：时钟，比如获取目前美国纽约的时间  

##### LocalDate
```java
// 指定特定的日期，调用of或parse方法返回该实例
LocalDate.of(2017, 07, 20);
LocalDate.parse("2017-07-20");
//哪一年;哪一月，月中哪一天，一周中第几天
int year = localDate.getYear();
Month month = localDate.getMonth(); 
int dayOfMonth = localDate.getDayOfMonth(); 
DayOfWeek dayOfWeek = localDate.getDayOfWeek(); 
//判断某个日期是星期几（周日对应7）
int value1 = LocalDate.parse("2020-09-28").getDayOfWeek().getValue();
// 一个月的天数
int lenth = localDate.lengthOfMonth(); 
// 是否是闰年
boolean leapYear = localDate.isLeapYear(); 
//获取当天日期、明天、昨天
LocalDate.now()
LocalDate tomorrow = LocalDate.now().plusDays(1);
LocalDate tomorrow = LocalDate.now().plusDays(-1);
LocalDate tomorrow = LocalDate.now().minusDays(1);//底层是plusDays方法
//从今天减去一个月，增加5天
LocalDate prevMonth = LocalDate.now().minus(1, ChronoUnit.MONTHS);
LocalDate after5Days=LocalDate.now().plus(5, ChronoUnit.DAYS);
//增加一年
localDateTime = localDateTime.plusYears(1);  
localDateTime = localDateTime.plus(1, ChronoUnit.YEARS);
//两个日期之间间隔多少天、月
long distance = ChronoUnit.DAYS.between(past, now);
long distance = ChronoUnit.MONTHS.between(past, now);
//生成一段时期中间的日期，包含起止日期，例子中间隔日期为8天，最后结果再list中，格式为yyyy-MM-dd
List<String> list = new ArrayList<String>();
LocalDate now=LocalDate.now();
LocalDate past = now.plusDays(-8);
long distance = ChronoUnit.DAYS.between(past, now);
Stream.iterate(past, d -> d.plusDays(1)).limit(distance + 1).forEach(f -> list.add(f.toString()));
//比较日期前后
isBefore和isAfter
//修改年为2019  
localDateTime = localDateTime.withYear(2020);  
//修改为2022  
localDateTime = localDateTime.with(ChronoField.YEAR, 2022);
//这个月的最后一天是几号、下个周末是几号
LocalDate localDate1 = LocalDate.now().with(firstDayOfYear());
```

##### LocalTime
```java
// 指定特定的时刻，调用of方法返回该实例
LocalTime localTime = LocalTime.of(22, 10, 59);
LocalTime localTime = LocalTime.parse("22:10:59");
//往后1小时
LocalTime nextHour = LocalTime.parse("15:02").plus(1, ChronoUnit.HOURS);
```

##### LocalDateTime
```java
// LocalDateTime包括LocalDate和LocalTime，可以根据后面两个进行初始化
LocalDateTime localDateTime = LocalDateTime.of(localDate, localTime);
//LocalDateTime 和 LocalDate, LocalTime 相互转换
LocalDate localDate1 = localDateTime.toLocalDate();
LocalTime localTime1 = localDateTime.toLocalTime();
//每天的开始和结束
LocalTime.MAX //对应23:59:59.999999999
LocalTime.MIN //对应00:00
```

##### Instant
```java
// 一个时间戳
Instant instant = Instant.now();
//获取秒数
long currentSecond = instant.getEpochSecond(); 
//获取毫秒数（这里不如直接用System.currentTimeMillis()方便）
long currentMilli = instant.toEpochMilli(); 
```

##### Duration
```java
// 一个时间段
Duration duration = Duration.between(past, now);
long toDays = duration.toDays(); // 这个时间段中有几天
long toHours = duration.toHours(); // 这个时间段中有几个小时
// 通过of创建时间段
Duration duration1 = Duration.of(5, ChronoUnit.DAYS);
```

##### Period
```java
// 以年月日来表示时间段:
Period period = Period.between(past, now);
period.getYears();
period.getMonths();
period.getDays();
```

##### 格式化
**<font color=red>DateTimeFormatter是线程安全的</font>**

```java
LocalDateTime now = LocalDateTime.now();
DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
System.out.println("默认格式化: " + now);
System.out.println("自定义格式化: " + now.format(dateTimeFormatter));
//字符串转LocalDateTime
LocalDateTime localDateTime = LocalDateTime.parse("2017-07-20 15:27:44", dateTimeFormatter);
//LocalDateTime按照自定义格式转字符串
DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");
String dateString = dateTimeFormatter.format(LocalDate.now());
```

##### 实操
- 将LocalDateTime字段以指定格式化日期的方式返回给前端  
在LocalDateTime字段上添加注解  
`@JsonFormat(shape=JsonFormat.Shape.STRING, pattern="yyyy-MM-dd HH:mm:ss")`  

- 对前端传入的日期进行格式化
在LocalDateTime字段上添加注解  
`@DateTimeFormat(pattern = "yyyy-MM-dd HH:mm:ss")`  

##### 新旧版本之间的转换
```java
//Date和Instant互相转换
Date date = Date.from(Instant.now());
Instant instant = date.toInstant();
//Date转换为LocalDateTime
LocalDateTime localDateTime = LocalDateTime.ofInstant(date.toInstant(), ZoneId.systemDefault());
//LocalDateTime转Date
Date date =
    Date.from(localDateTime.atZone(ZoneId.systemDefault()).toInstant());
//LocalDate转Date
Date date =
 Date.from(LocalDate.now().atStartOfDay().atZone(ZoneId.systemDefault()).toInstant());
```

### 总结
&emsp;&emsp;Java8之后的时间类已经具备了很多比较人性化的API了，所以在新项目中可以采用新的时间类，老项目中还是用以前的类，而且老项目中一般会有已有的时间处理工具类。



