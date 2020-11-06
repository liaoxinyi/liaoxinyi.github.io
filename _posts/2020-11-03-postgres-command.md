---
layout:     post
title:      "PostgresSQL的常用指令"
subtitle:   "pg数据库的一些杂项指令，不定期更新"
date:       2020-11-03
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/pgSQL-index.bmp"
tags:
    - PostgresSQL
---
> 来源于网络，不定期更新

### 占用大小查询
```sql
-- 查询单个数据库大小
select pg_size_pretty(pg_database_size('postgres')) as size;
-- 查询所有数据库大小
select datname, pg_size_pretty (pg_database_size(datname)) AS size from pg_database;
-- 查询单个表大小
select pg_size_pretty(pg_relation_size('mytab')) as size;
 

-- 查询所有表大小
select relname, pg_size_pretty(pg_relation_size(relid)) as size from pg_stat_user_tables; 
-- 查询单个表的总大小，包括该表的索引大小
select pg_size_pretty(pg_total_relation_size('tab')) as size;
-- 查询所有表的总大小，包括其索引大小
select relname, pg_size_pretty(pg_total_relation_size(relid)) as size from pg_stat_user_tables;

-- 查询单个索引大小
select pg_size_pretty(pg_relation_size('myindex')) as size;
-- 查询单个表空间大小
select pg_size_pretty(pg_tablespace_size('pg_default')) as size;
-- 查询所有表空间大小
select spcname, pg_size_pretty(pg_tablespace_size(spcname)) as size from pg_tablespace;
-- 或
select spcname, pg_size_pretty(pg_tablespace_size(oid)) as size from pg_tablespace;
```

### 清空表内容
```sql
//当表没有其他关系时
TRUNCATE TABLE 表名;
//当表中有外键时，要用级联方式删所有关联的数据
TRUNCATE TABLE 表名 CASCADE;
```

### 用户密码相关
- 创建用户：postgres# CREATE USER xxxx1 WITH PASSWORD 'xxxx';
- 创建数据库：postgres# CREATE DATABASE xxxx2;
- 赋权：postgres# GRANT ALL PRIVILEGES ON DATABASE xxxx2 to xxxx1;
- 修改密码：alter user 用户 with password '新密码';
- 实体类中各种类型对应的pgSQL：  
**Float类型**：字段名  double precision  
**Timestamp类型**：字段名  timestamp(0) without time zone DEFAULT NULL::timestamp without time zone  
```sql
--如果存在则删除表
DROP TABLE IF EXISTS "public"."tb_user_password";
--建表
CREATE TABLE "public"."tb_user_password" (
  "pk_user_password_id" int4 NOT NULL DEFAULT NULL,
  "out_time" int8 NOT NULL DEFAULT NULL,
  "sid" varchar(255) COLLATE "pg_catalog"."default" NOT NULL DEFAULT NULL,
  "user_name" varchar(255) COLLATE "pg_catalog"."default" NOT NULL DEFAULT NULL,
  "login_date" timestamp(6) NOT NULL DEFAULT NULL
);
--添加描述
COMMENT ON COLUMN "public"."tb_user_password"."pk_user_password_id" IS '主键';
COMMENT ON COLUMN "public"."tb_user_password"."out_time" IS '过期时间';
COMMENT ON COLUMN "public"."tb_user_password"."sid" IS 'sid';
COMMENT ON COLUMN "public"."tb_user_password"."user_name" IS '用户名';
--主键
ALTER TABLE "public"."tb_menu" ADD CONSTRAINT "tb_menu_pkey" PRIMARY KEY ("pk_id");
--主键自增策略
DROP SEQUENCE IF EXISTS "tb_menu_seq";
CREATE SEQUENCE tb_menu_seq INCREMENT 1 MINVALUE 1 NO MAXVALUE START 1 CACHE 20;
--初始化数据
INSERT INTO "tb_user_role"("pk_id", "role_id", "user_id") VALUES (nextval('tb_user_role_seq'), 1, 1);
--添加字段
ALTER TABLE public.imap_place_info ADD COLUMN IF NOT EXISTS  metro_line_id int8 NOT NULL DEFAULT 1;
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
--批量更新
update t_inspect 
set result_value=tmp.result_value,
result_varchar=tmp.result_varchar,
modify_user = tmp.modify_user,
modify_date = tmp.modify_date
from (values (67,7,7,'test',LOCALTIMESTAMP),(68,8,8,'test',LOCALTIMESTAMP)) 
as tmp (id_inspect,result_value,result_varchar,modify_user,modify_date) 
where t_inspect.id_inspect = tmp.id_inspect; 
```
