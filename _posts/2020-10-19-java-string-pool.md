---
layout:     post
title:      "new String(\"字面量\")时发生了什么？"
subtitle:   "字符串常量池详细解读"
date:       2020-10-19 23:59:00
author:     "ThreeJin"
header-mask: 0.3
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/string-pool.bmp"
tags:
    - Java
---
> 来源于知乎的一次问题讨论：Java 中new String("字面量") 中 "字面量" 是何时进入字符串常量池的?

### 前言
&emsp;&emsp;在回顾完JVM的相关知识后，分清楚了三种常量池：字符串常量池、Class常量池与运行时常量池的区别。那么问题来了，当虚拟机中开始运行到new String("字面量")这行代码的时候（准确的说不是运行到这行代码，而是字节码被加载到内存中后，发生的一列事情），会是什么样的情形呢？所以，这里记录一下在复盘这次讨论的过程中时，其他前辈们针对这个问题的一些看法
### 三种常量池的关系
1. Class文件中的常量池主要存放编译期生成的两大类常量（字面量和符号引用）  
**也就是说，用双引号引起来的字符串字面量都会进入Class文件中的常量池**  
2. 在类被加载后，Class文件中的常量池进入方法区后就变成了运行时常量池  
3. 全局字符串常量池只是一个纯运行时的结构，实质上是一个HashSet<String>，只存储对java.lang.String实例的引用  

**<font color=red>一般我们说一个字符串进入了全局的字符串常量池其实是说在字符串常量池的StringTable中保存了对它的引用，反之，如果说没有在其中就是说StringTable中没有对它的引用</font>**

### String#intern()
&emsp;&emsp;JDK7中，如果常量池中已经有了这个字符串，那么直接返回常量池中它的引用，如果没有，那就将它的引用保存一份到字符串常量池，然后直接返回这个引用，重点就是**这个方法是有返回值的，而且返回的是引用**

### 问题复盘
#### 复盘代码
```java
class NewTest1{
    public static String s1="static";  // 第一句
    public static void main(String[] args) {
        String s2=new String("he")+new String("llo"); //第二句
        s1.intern();   //将 堆中新建的对象"hello" 存入字符串常量池
        String s3="hello";  //第三句
        System.out.println(s2==s3);//输出是true。
    }
}
```
#### 复盘分析
&emsp;&emsp;首先，编译器在完成代码编译之后，"static" "he" "llo" "hello"都会进入Class的常量池。其中，**因为类加载过程中的解析阶段是lazy的**，所以是不会立马创建实例，更不会立马驻留字符串常量池  
&emsp;&emsp;那么，既然是lazy的，总有一个时刻或者一个条件来触发它。这就是那16个用于操作符号引用的字节码指令（比如Idc指令），以Idc指令为例：这个指令的作用是将int、float或String型常量值从常量池中推送至栈顶。具体实现过程是让虚拟机到当前类的运行时常量池（runtime constant pool，HotSpot VM里是ConstantPool + ConstantPoolCache）去查找该index对应的项，如果该项尚未resolve则resolve之，并返回resolve后的内容。在遇到String类型常量时，resolve的过程如果发现StringTable已经有了内容匹配的java.lang.String的引用，则直接返回这个引用，反之，如果StringTable里尚未有内容匹配的String实例的引用，则会在Java堆里创建一个对应内容的String对象，然后在StringTable记录下这个引用，并返回这个引用出去。可见，ldc指令是否需要创建新的String实例，全看在第一次执行这一条ldc指令时，StringTable是否已经记录了一个对应内容的String的引用  
&emsp;&emsp;但是，要注意上面那段代码中的“static”和其他三个不一样，它是静态的。所以在类加载过程中的初始化阶段，会为静态变量指定初始值，也就是要把“static”赋值给s1。具体的操作就是，先ldc指令把它放到栈顶，然后用putstatic指令完成赋值。注意，ldc指令，根据上面说的，会创建"static"字符串对象，并且会保存一个指向它的引用到字符串常量池  
&emsp;&emsp;运行main方法后，首先是第二句，一样的，要先用ldc把"he"和"llo"送到栈顶，换句话说，会创建他俩的对象，并且会保存引用到字符串常量池中。然后有个＋号，内部是创建了一个StringBuilder对象，一路append，最后调用StringBuilder对象的toString方法得到一个String对象（内容是hello，注意这个toString方法会new一个String对象），并把它赋值给s1。注意啊，此时并没有把hello的引用放入字符串常量池  
&emsp;&emsp;然后是第三句，intern方法一看，字符串常量池里面没有，它会把上面的这个hello对象的引用保存到字符串常量池，然后返回这个引用，但是这个返回值我们并没有使用变量去接收，所以没用  
&emsp;&emsp;第四句，字符串常量池里面已经有了，直接用嘛  
&emsp;&emsp;第五句，已经很明显了  
```java
class NewTest2{
    public static void main(String[] args) {
        String s1=new String("he")+new String("llo");
        String s2=new String("h")+new String("ello");
        String s3=s1.intern();
        String s4=s2.intern();
        System.out.println(s1==s3);
        System.out.println(s1==s4);
    }
}
```
&emsp;&emsp;类加载阶段，什么都没干  
&emsp;&emsp;然后运行main方法，先看第一句，会创建"he"和"llo"对象，并放入字符串常量池，然后会创建一个"hello"对象，没有放入字符串常量池，s1指向这个"hello"对象  
&emsp;&emsp;第二句，创建"h"和"ello"对象，并放入字符串常量池，然后会创建一个"hello"对象，没有放入字符串常量池，s2指向这个"hello"对象  
&emsp;&emsp;第三句，字符串常量池里面还没有，于是会把s1指向的String对象的引用放入字符串常量池（换句话说，放入池中的引用和s1指向了同一个对象），然后会把这个引用返回给了s3，所以s3==s1是true  
第四句，字符串常量池里面已经有了，直接将它返回给了s4，所以s4==s1是true  
>以上复盘来自知乎讨论中"木女孩"这位大佬的见解，[直达电梯](https://www.zhihu.com/question/55994121)

### 总结
- 用双引号引起来的字符串字面量都会进入Class文件中的常量池
- String#intern()这个方法是有返回值的，而且返回的是引用
- 因为类加载过程中的解析阶段是lazy的，所以字符串字面量在进入Class的常量池后不会立马创建实例，更不会立马驻留字符串常量池，只有在那16个操作符号引用的字节码指令执行前，才会触发真正解析的工作
