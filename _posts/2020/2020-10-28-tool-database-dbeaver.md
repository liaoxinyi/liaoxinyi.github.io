---
layout:     post
title:      "数据库利器——DBeaver"
subtitle:   "告别Navicat"
color:      "white"
date:       2020-10-28
author:     "ThreeJin"
header-mask: 0.6
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/idea.jpg"
tags:
    - 工具
---
> 告别Navicat，转战DBeaver，非常强大易用的数据库管理和开发工具。部分资料来源于[不剪发的Tony老师](https://blog.csdn.net/horses/article/details/89683422)，不定期更新

### 前言
DBeaver 是一个基于 Java 开发，免费开源的通用数据库管理和开发工具，使用非常友好的 ASL 协议。它可以运行在各种操作系统上，包括：Windows、Linux、macOS 等。DBeaver 采用 Eclipse 框架开发，支持插件扩展，并且提供了许多数据库管理工具：ER 图、数据导入/导出、数据库比较、模拟数据生成等  
DBeaver 通过 JDBC 连接到数据库，可以支持几乎所有的数据库产品，包括：MySQL、PostgreSQL、MariaDB、SQLite、Oracle、Db2、SQL Server、Sybase、MS Access、Teradata、Firebird、Derby 等等。**商业版本**更是可以支持各种 NoSQL 和大数据平台：MongoDB、InfluxDB、Apache Cassandra、Redis、Apache Hive 等  
### 下载与安装
##### 下载
DBeaver 社区版可以通过[官方网站](https://dbeaver.io/download/)或者 [Github](https://github.com/dbeaver/dbeaver/releases) 进行下载
##### 安装
DBeaver支持中文，但DBeaver的运行依赖于JRE

### 进阶使用
常规的建立、连接和操作数据库的步骤就不再赘述了，这里主要讲一下一些提高开发效率的功能
##### DDL
右键某个表--生成SQL---DDL：可以生成对应的SQL语句，比手写要来得方便得多（虽然长期这样会降低自己手写SQL的能力，但是来得快啊，真香系列）
##### 字段排序筛选
- 不关心字段数据时  
如果只想看字段有哪些，可以直接在属性看，字母排序啥的都有  
- 想要看数据时  
如果字段太多，想要看某个字段的内容时，把所有字段排个序：【数据】--【右上角工具栏】--【自定义过滤】：此时可以进行筛选、排序等操作

### 总结
- 工具只是工具，最重要的还是能力和习惯
- 有计划还是多练习SQL，使用工具的时候，还是不要忘了最基础的东西
