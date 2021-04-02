---
layout:     post
title:      "ORM IN CHINA-mybatis-01"
subtitle:   "Springboot集成mybatis、OGNL标签、一些基础坑"
date:       2021-03-30
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/mybatis-bk-01.jpg"
tags:
    - Java
    - ORM
    - mybatis

---
> 资料来源于网络上各位前辈（古时的风筝、Java团长、微笑很纯洁、阿进得写字台等）

### 前言
终于到了沉淀时间了，自开年返工后，一直都在忙项目，无暇顾及整理一些东西。历时一个月的闭关开发，逐渐理解为什么大家都在说工作之后就很难有时间再进行二次学习。每天的加班，回到酒店后，哪还有心思来整理一天的所得，只想到头就睡。废话不多说，直接开搞。  
之前一年多里面，提及ORM，基本上我都只会想去使用JPA，不会想到说试试mybatis。除了来自Hibernate的熟悉感外，更多的是图便捷。一来懒惰思维在作怪，二来每次都是时间紧迫，mybatis的一些零碎让我望而却步。这次，因为项目前半部分已经采用了mybatis了，所以正好逼着自己开始重新拾起mybatis。以前玩mybatis的时候，都还要自己写很多xml，写很多mapper，现在在IDEA下，有了插件后，简直不要爽歪歪。  
说到mybatis，一般都会拿它和JPA进行对比，同为ORM框架，mybatis在国内享有很高的普及度，但仅仅局限于国内而已。国外，大部分都还是JPA的粉丝，从论坛上的一些提问也能看出来，mybatis的下面大部分都是国人在进行讨论，mybatis-plus也是国人在原来基础上的封装加强（无侵入）。怎么说呢？心底里还是会觉得有那么一丝淡淡的不服气，说到底，还是我们自己技不如人。  

### Springboot集成mybatis
##### mybatis-spring-boot-starter

之所以先提这个，是因为myBatis看spring boot这么火热也开发出一套解决方案来凑凑热闹,但这一凑确实解决了很多问题，使用起来确实顺畅了许多。mybatis-spring-boot-starter主要有两种解决方案，一种是使用注解解决一切问题，一种是简化后的老传统。这里还是比较推荐使用简化版的老传统：Mapper+xml的方式，原因就在于开发时，往往会复制代码中的sql进行调试，如果采用注解的方式，复制粘贴出来的代码需要修改很大部分才能在sql编辑器中运行。如果采用xml的方式，则几乎不用更改sql，因为xml中本身就是sql+OGNL表达式。 

#####Pom依赖
```xml
<!--Springboot-->
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
	<!--此时2021我的版本是2.0.1-->
	<version>2.0.1</version>
</dependency>

<!--分页查询工具-->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper-spring-boot-starter</artifactId>
</dependency>

<!--编译时候需要注意的-->
<build>
		<resources>
			<!-- maven项目中src源代码下的xml等资源文件编译进classes文件夹，
              注意：如果没有这个，它会自动搜索resources下是否有mapper.xml文件，
              如果没有就会报org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): com.pet.mapper.PetMapper.selectByPrimaryKey-->
			<resource>
				<directory>src/main/java</directory>
				<includes>
					<include>**/*.xml</include>
				</includes>
			</resource>
			<!--将java目录下的配置文件编译进classes文件  -->
			<resource>
				<directory>src/main/java</directory>
			</resource>
		</resources>

</build>
```  

上面的pageHelper，配合mybatis使用，分页查询简单快捷  

##### application.properties
```java
mybatis.type-aliases-package=com.neo.entity

spring.datasource.driverClassName = com.mysql.jdbc.Driver
spring.datasource.url = jdbc:mysql://localhost:3306/test1?useUnicode=true&characterEncoding=utf-8
spring.datasource.username = root
spring.datasource.password = root
```  

springboot会自动加载spring.datasource.\*相关配置，数据源就会自动注入到sqlSessionFactory中，sqlSessionFactory会自动注入到Mapper中，直接拿起来使用就行了  

##### @MapperScan
这里可以采用两种方式：  
- 在启动类中添加对mapper包扫描`@MapperScan("com.xxx.mapper")`  
- 直接在Mapper类上面添加注解`@Mapper`

##### myBatis-generator
有了插件后，mybatis最为繁琐的mapper、xml等等配置都变得无比丝滑，而且附带了常用的CRUD操作、

### 一些基础坑
##### 主键自增的问题
- 问题描述

在使用mybatis-generator插件生成的save方法的时候，xml中的配置如下：
```xml
<insert id="insert" keyColumn="pk_id" keyProperty="pkId" parameterType="com.xxx.entity.XxxEntity" useGeneratedKeys="true">
    some sql
</insert>
```

- 问题解决

但是在实际运行的时候，发现执行保存的时候会报错，问题在于`useGeneratedKeys`这一属性和建表时的自增序列选择冲突了，这里贴一个合适的建表自增序列：

```sql
DROP SEQUENCE IF EXISTS "tb_xxx_seq" CASCADE;
CREATE SEQUENCE tb_xxx INCREMENT 1 MINVALUE 1 NO MAXVALUE START 1 CACHE 20;
DROP TABLE IF EXISTS "tb_xxx";
CREATE TABLE public.tb_xxx (
	pk_id int4 primary key DEFAULT nextval('tb_xxx_seq'::regclass)
);
```

##### 分页查询的问题

如果使用的pageHelper的插件的话，可以采用如下的方式进行分页查询：

```java
PageHelper.startPage(pageNum, pageSize);
// to do query operation and return xxxEntities {@code List<XxxEntity>}
PageInfo<XxxEntity> pageInfo = new PageInfo<>(xxxEntities);
//pageInfo.getTotal()
```

### mybatis中的OGNL
##### 常用标签

|元素|作用|备注|  
|----|----|----|  
|if|判断语句|单条件分支，必须结合test使用|  
|choose(when、otherwise)|相当于java中的switch，choose 为 switch，when 为 case，otherwise 则为 default|多条件分支|  
|trim(where、set)|辅助元素|用于处理SQL的拼接问题|  
|foreach|循环语句|批量问题经常会遇到|  
|bind|创建一个变量并绑定到上下文中|用于兼容不同的数据库，防止SQL注入等|  

##### if标签
- 基本的标签格式

**if标签的test属性值是一个符合OGNL的表达式，表达式可以是true或false。如果表达式返回的是数值，则0为false,非0为true**

```xml
<if test="xxx != null">
    表达式内容
</if>
```

- 条件查询的时候

```xml
<select id="selectByStudentSelective" resultMap="BaseResultMap" parameterType="com.homejim.mybatis.entity.Student">
    select
    <include refid="Base_Column_List" />
    from student
    where 1=1
    <if test="name != null and name !=''">
      and name like concat('%', #{name}, '%')
    </if>
    <if test="sex != null">
      and sex=#{sex}
    </if>
</select>
```

其中`where 1=1`往往用于在多条件拼接的时候会使用，这样方便后续的条件都可以用`and`来进行连接

- UPDATE的时候

往往通过test判断属性是否为空来选择性更新

##### choose标签
最开始的时候，我其实有点模糊choose和if的关系。但是仔细想一想，如果某时候我并不想应用所有的条件，而只是想从多个选项中选择一个时，if标签就不满足要求了。因为只要test中的表达式为 true，就会执行 if 标签中的条件。这个时候就需要choose 元素了，if标签是与(and)的关系，而 choose 是或(or)的关系。

**choose标签是按顺序判断其内部when标签中的test条件出否成立，如果有一个成立，则 choose 结束，当 choose 中所有 when 的条件都不满则时，则执行 otherwise 中的sql**

##### where标签
where标签只会在至少有一个子元素的条件返回 SQL 子句的情况下才去插入`WHERE`子句。而且，若语句的开头为`AND`或`OR`，where 元素也会将它们去除。这意味着可以过滤掉条件语句中的第一个and或or关键字，但是**where标签只能去除第一个条件中出现的前置（前置很重要！！！）and 关键字。像以下情况，where就无能为力了:**

```xml
<select id="selectIfCondition" resultType="com.heiketu.testpackage.pojo.Product">
    SELECT
        prod_id prodId,
        vend_id vendId,
        prod_name prodName,
        prod_desc prodDesc
    FROM Products
    <where>
        <if test="prodId != null and prodId != ''">
            prod_id = #{prodId} AND
        </if>

        <if test="prodName != null and prodName != ''">
            prod_name = #{prodName} AND
        </if>
    </where>
</select>
```

如果要解决上面的问题，需要使用`trim`标签

##### set标签
同where标签功能类似，`where`用去去除第一个条件中出现的`and`或者`or`前缀，那么`set`标签就是去除最后一个更新字段语句中出现的`,`后缀

##### trim标签
mybatis的trim标签一般用于去除sql语句中多余的and关键字，逗号，或者给sql语句前拼接 “where“、“set“以及“values(“ 等前缀，或者添加“)“等后缀，可用于选择性插入、更新、删除或者条件查询等操作。

|属性|描述|  
|----|----|  
|prefix|当 trim 元素包含有内容时，给sql语句拼接的前缀|  
|suffix|当 trim 元素包含有内容时，给sql语句拼接的后缀|  
|prefixOverrides|当 trim 元素包含有内容时，去除sql语句前面的关键字或者字符，该关键字或者字符由prefixOverrides属性指定，假设该属性指定为"AND"，当sql语句的开头为"AND"，trim标签将会去除该"AND"|  
|suffixOverrides|当 trim 元素包含有内容时，去除sql语句后面的关键字或者字符，该关键字或者字符由suffixOverrides属性指定|  

比如上面的`where`标签就可以改写为：

```xml
<trim prefix="where" prefixOverrides="AND |OR">
</trim>
```

##### foreach标签

|属性|描述|  
|----|----|  
|collection|需要遍历的数据类型|  
|item|即从迭代的对象中取出的每一个值|  
|index|索引的属性名。 当迭代的对象为 Map 时， 该值为 Map 中的 Key|  
|open|循环开头的字符串|  
|close|循环结束的字符串|  
|separator|每次循环的分隔符|  

- collection的设置问题
1. 只有一个数组参数或集合参数

默认情况： 集合collection=list， 数组是collection=array
推荐： 使用 `@Param` 来指定参数的名称， 如我们在参数前`@Param("ids")`， 则就填写 `collection=ids`

2. 多参数

多参数请使用 `@Param` 来指定， 否则SQL中会很不方便

3. 参数是Map

指定为 Map 中的对应的 Key 即可。 其实上面的 `@Param` 最后也是转化为 Map 的。

4. 参数是对象

正常使用属性即可

##### bind标签
通过 OGNL 表达式去定义一个上下文的变量，比如下面这个例子：

```xml
<if test="name != null and name !=''">
      and name like concat('%', #{name}, '%')
    </if>
```

在 MySQL 中， 该函数支持多参数， 但在 Oracle 中只支持两个参数。 那么可以使用 bind 来让该 SQL 达到支持两个数据库的作用:

```xml
<if test="name != null and name !=''">
     <bind name="nameLike" value="'%'+name+'%'"/>
     and name like #{nameLike}
</if>
```

### 总结
无论使用哪一个orm，最重要的还是数据库基础，注入事务、索引、sql优化等，所以后续需要加强这方面的细化总结。