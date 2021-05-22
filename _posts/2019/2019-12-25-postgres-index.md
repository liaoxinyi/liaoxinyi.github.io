---
layout:     post
title:      "PostgresSQL的索引"
subtitle:   "针对pg数据库中索引的一些研究"
date:       2019-12-25
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/pgSQL-index.bmp"
tags:
    - PostgresSQL
---
> 来源于[PostgreSQL中文文档](https://www.yiibai.com/manual/postgresql/index.html)，不定期更新

### 简介
在这之前，一直对于数据库的索引停留在主键索引的阶段，比如我们经常执行这样的语句：
```sql
SELECT content FROM test1 WHERE id = XXX;
```
当id字段上没有索引时，数据库会一行一行去找结果，但是如果id上有个索引，那数据库就可以用一种更高效的方式定位结果

### 特点
- 一旦一个索引被创建，就不再需要进一步的干预：系统会在表更新时更新索引，而且会在它觉得使用索引比顺序扫描表效率更高时使用索引
- 可能需要定期地运行ANALYZE命令来更新统计信息以便查询规划器能做出正确的决定
- 在一个大表上创建一个索引会耗费很长的时间。默认情况下，PostgreSQL允许在索引创建时并行地进行读（SELECT命令），但写（INSERT、UPDATE和DELETE）则会被阻塞直到索引创建完成，如果需要则进行并发构建索引

### 常见类型
- B-tree、Hash、GiST、SP-GiST、GIN 和 BRIN  
每一种索引类型使用了一种不同的算法来适应不同类型的查询。 默认情况下，CREATE INDEX 命令创建的是B-tree 索引
- PostgreSQL的查询规划器会在任何一种涉及到以下操作符的已索引列上考虑使用B-tree索引：  
**<  <=  = >= >**  
- 处理模糊查询时，只有当模糊片段处在开头时（'foo%'这样）才会使用B-tree
- Hash索引只能处理简单等值比较。不论何时当一个索引列涉及到一个使用了=操作符的比较时，查询规划器将考虑使用一个Hash索引，但是因为Hash索引经常需要被重建（发生数据库崩溃等问题后）所以一般不建议使用

### 多列索引
只有 B-tree、GiST、GIN 和 BRIN索引类型支持多列索引， 最多可以指定32个列（该限制可以在源代码文件 pg_config_manual.h中修改， 但是修改后需要重新编译PostgreSQL）

### 组合索引
- 只有查询子句中在索引列上使用了索引操作符类中的操作符并且通过AND连接时才能使用单一索引。例如，给定一个(a, b) 上的索引，查询条件WHERE a = 5 AND b = 6可以使用该索引，而查询WHERE a = 5 OR b = 6不能直接使用该索引
- 多列索引和组合索引的取舍  
例如，如果我们的查询中有时只涉及到列x，有时候只涉及到列Y，还有时候会同时涉及到两列，我们可以选择在x和y上创建两个独立索引然后依赖索引组合来处理同时涉及到两列的查询。我们当然也可以创建一个(x, y)上的多列索引。当查询同时涉及到两列时，该索引会比组合索引效率更高，但它在只涉及到y的查询中几乎完全无用，因此它不能是唯一的一个索引。一个多列索引和一个y上的独立索引的组合将会工作得很好。多列索引可以用于那些只涉及到x的查询，尽管它比x上的独立索引更大且更慢。最后一种选择是创建所有三个索引，但是这种选择最适合表经常被执行所有三种查询但是很少被更新的情况。如果其中一种查询要明显少于其他类型的查询，我们可能需要只为常见类型的查询创建两个索引

### 唯一索引
- 只有B-tree能够被声明为唯一
- 当一个索引被声明为唯一时，索引中不允许多个表行具有相同的索引值。空值被视为不相同。一个多列唯一索引将会拒绝在所有索引列上具有相同组合值的表行
- PostgreSQL会自动为定义了一个唯一约束或主键的表创建一个唯一索引。该索引包含组成主键或唯一约束的所有列（可能是一个多列索引），它也是用于强制这些约束的机制
- **不需要手工在唯一列上创建索引，那样做只会重复建立自动创建的索引**

### 其他
- 要使索引发挥作用，查询条件中的列必须要使用适合于索引类型的操作符，使用其他操作符的子句将不会被考虑使用索引
- 索引和ORDER BY  
默认情况下，B-tree索引将它的项以升序方式（ASC）存储，并将空值放在最后。可以在创建B-tree索引时通过ASC、DESC、NULLS FIRST和NULLS LAST选项来改变索引的排序

### 命令实践
```sql
--在id字段上创建一个索引
--索引的名字test1_id_index可以自由选择，建议选择一个表征索引用途的名字
--移除索引采用DROP INDEX
--索引可以随时被创建或删除
CREATE INDEX test1_id_index ON test1 (id);
--创建多列索引
CREATE INDEX test2_mm_index ON test2 (major, minor);
--创建带排序的索引，NULLS FIRST是ORDER BY DESC的默认情况
--也可以尝试将索引定义为(x ASC, y DESC)或者(x DESC, y ASC)
CREATE INDEX test2_info_nulls_low ON test2 (info NULLS FIRST);
CREATE INDEX test3_desc_index ON test3 (id DESC NULLS LAST);
--声明索引的唯一性
CREATE UNIQUE INDEX name ON table (column [, ...]);
```
