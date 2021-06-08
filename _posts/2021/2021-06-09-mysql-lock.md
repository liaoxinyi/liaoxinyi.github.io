---
layout:     post
title:      "又见MySQL-03"
subtitle:   "各种锁"
date:       2021-06-09
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-mysql-bk-02.jpg"
tags:
    - MySQL

---
> 资料来源于网络上各位前辈（Hollis等）

### 前言
之前一直总结的各类锁都是针对线程并发的，其实不管是什么锁，归根结底都是为了解决多线程或者多进程并发去修改某一个资源的时候带来的数据不安全问题
### 锁的粒度

##### 表锁
表锁由 MySQL Server 实现，行锁则是存储引擎实现，不同的引擎实现的不同。在 MySQL 的常用引擎中 InnoDB 支持行锁，而 MyISAM 则只能使用 MySQL Server 提供的表锁

表锁由 MySQL Server 实现，一般在执行 DDL 语句时会对整个表进行加锁，比如说 ALTER TABLE 等操作。除此之外，在执行 SQL 语句时，也可以明确指定对某个表进行加锁

除了使用 unlock tables 显示释放锁之外，会话持有其他表锁时执行lock table 语句会释放会话之前持有的锁；会话持有其他表锁时执行 start transaction 或者 begin 开启事务时，也会释放之前持有的锁

- **共享读锁(table read lock)**  
对表的读操作不会阻塞其他用户对同一表的读操作，但会阻塞对同一表的写请求  
- **独占写锁(table write lock)**  
对表的写操作，会阻塞其他用户对同一表的读和写操作  
- **查询表级锁竞争情况**  
可以通过检查table_locks_waited和table_locks_immediate状态变量来分析系统上的表锁定争夺：如果table_lock_waited的值比较高，则说明存在较严重的表级锁争用情况

##### 行锁
行锁由存储引擎实现，InnoDB 支持，而 MyISAM 不支持。行锁的优点是锁粒度小，发生锁冲突概率小，并发度高，缺点是开销大、加锁慢，并且可能产生死锁

**<font color=red>InnoDB 行锁是通过索引项加锁来实现的</font>**，只有通过索引条件检索数据，才能锁住指定的索引记录，否则将使用行锁锁住全部数据

表级锁适合查询多、更新少的场景，行级锁适合按索引更新频率高的场景。**InnoDB 默认使用行级锁**

### 锁的模式
##### 共享锁和排它锁
**共享锁和排它锁都是行级锁**

- **Shared Lock (S 锁)**  
`SELECT * FROM table_name WHERE ... LOCK IN SHARE MODE`  
又称读锁。允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁。若事务1对记录A加上S锁，则事务1可以读取A但不能修改A，其他事务只能再对A加S锁，不能加X锁。这就保证了其他事务可以读A，但在事务1释放A上的S锁之前不能对A做任何的修改  
- **Exclusive Lock (X 锁)**  
`SELECT * FROM table_name WHERE ... FOR UPDATE`  
又称写锁。获取排他锁的事务可以更新数据，阻止其他事务获取相同数据集的共享读锁和排他写锁。若事务1对记录A加了X锁，则事务1可以读A，也可以修改A，其他事务不能对A加任何锁，知道事务1释放A上的锁  
注意：对于共享锁很好理解，就是多个事务只能读取数据不能修改数据。但是排他锁容易错误地理解成：如果一个事务锁住一行数据后，其他事务不能读物和修改该行数据。其实排他锁指的是一个事务对一条记录加上排他锁，其他事务不能对该记录加其他的锁。innoDb引擎默认的update,delete和insert会自动给涉及的数据加上排他锁，select语句默认不会加任何锁。所以加过排他锁的数据行在其他事务中不能修改数据，也不能通过for update加排他锁或者lock in share mode加共享锁，但是可以直接通过select...from...的方式查询数据，因为普通的查询没有任何锁机制

##### 意向锁
**意向锁属于表锁**

- **Intention Shared Lock (IS)**  
意向共享锁，也称为意向读锁。意向共享锁表示有事务打算在行记录上加共享锁，在事务获取行 S 锁前，必须先获得 IS 锁或更高级别的锁  
- **Intention Exclusive Lock (IX)**    
意向排它锁，也称为意向写锁。意向排它锁表示有事务打算在行记录上加排它锁，在事务获取行 X 锁前，必须先获 IX 锁  

**意向锁之间不会发生冲突，但共享锁、排它锁、意向锁之间会发生冲突**
##### 自增锁
`AUTO-INC Locks`，自增锁，它是一种特殊的表锁。当表有设置自增 `auto_increment` 列，在插入数据时会先获取自增锁，其它事务将会被阻塞插入操作，自增列 +1 后释放锁，如果事务回滚，自增值也不会回退，所以自增列并不一定是连续自增的

### 行锁的分类
##### 记录锁(Record Locks)
记录锁是索引记录的锁定  
`SELECT a FROM t WHERE a = 15 FOR UPDATE`  
上面sql对索引记录 15 进行锁定，防止其它事务插入、删除、更新值为 15 的记录行

原理是**通过索引加锁**，如果列没有设置索引，则将使用聚簇索引，如果没有人为指定聚簇索引，MySQL 会自动建立一个聚簇索引。

##### 间隙锁(Gap Locks)
间隙锁是对索引记录之间的间隙的锁定。锁定的对象是键值在条件范围内但并不存在的记录，叫做间隙（gap）  
`SELECT a FROM t WHERE a > 15 and a < 20 FOR UPDATE`  
其中，a真实存在的值为 1、2、5、10、15、20，则间隙锁会将 15,20 中的间隙锁住

**作用就是为了防止其他事务的插入，在 RR（可重复读）级别下解决了幻读的问题**

##### 临键锁(Next-Key Lock)
临键锁其实就是记录锁和间隙锁的合集  
`SELECT a FROM t WHERE a > 15 FOR UPDATE`  
其中，a真实存在的值为 1、2、5、10、15、20，则将 (15,20]、(20, +∞] 的中 15、20 及其间隙锁住

这里有一个很形象的例子：  
![](https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/java-mysql-03-01.jpg)  
<center>几种行锁的分类</center>  

##### 插入意向锁(Insert Intention Locks)
插入意向锁是一种特殊的间隙锁，只有在执行`INSERT`操作时才会加锁，各个插入意向锁彼此之间不冲突，可以向一个间隙中同时插入多行数据，但插入意向锁与间隙锁是冲突的，当有间隙锁存在时，插入语句将被阻塞，正是这个特性解决了幻读的问题
### 死锁
类似于多线程里面的死锁：  

```sql
// 第一步：事务 1 执行
UPDATE test_lock SET money = 1001 WHERE id = 5;
// 第一步：事务 2 执行
UPDATE test_lock SET money = 1001 WHERE id = 12;
// 第二步：事务 1 执行
UPDATE test_lock SET money = 1001 WHERE id = 12;
// 第二步：事务 2 执行
UPDATE test_lock SET money = 1001 WHERE id = 5;
```

第一步执行后，事务 1 对 5 持有 X 锁，事务 2 对 12 持有 X 锁。执行第二步时，事务 1 在等待事务 2 对 12 的释放，事务 2 在等待事务 1 对 5 的释放，由此产生了死锁





