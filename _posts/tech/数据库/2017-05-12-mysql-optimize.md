---
layout: post
title: SQl语句优化技巧
category: 数据库
tags: mysql;优化技巧
keywords: 
description:
---

---
### 1.通过show status 命令整体了解数据库运行的状况
> show [session|global] status 命令 可以提供服务器的状态信息。
global 级（自数据库上次启动至今）的统计结果，session级（当前连接）的统计结果。

show status like 'Com_%'; 
Com_xxx表示每个xxx语句执行的次数，我们通常比较关心的是以下几个统计参数。
>* Com_select: 执行select 操作的次数，一次查询只累加1.
>* Com_insert :执行insert 操作的次数，对于批量插入的insert操作，只累加一次。
>* Com_update:执行Update 操作的次数。
>* Com_delete:执行delete 操作的次数。

上面这些参数对于所有存储引擎的表操作都会进行累计。下面这几个参数只是针对innoDB 存储引擎的，累加的算法也略有不同。

>* Innodb_rows_read: select 查询返回的行数。

>* Innodb_rows_inserted:执行insert 操作插入的行数。

>* innodb_rows_updated:执行update操作更新的行数。

>* Innodb_rows_deleted:执行delete操作删除的行数。

通过以上几个参数，可以很容易地了解到当前数据库的应用是以插入更新为主还是以查询操作为主，以及各种类型的SQl 大致执行的比例是多少。对于跟新操作的计数，是对执行次数的计数，不论提交还是回滚都会进行累加。

对于事务型的应用，通过Com_commit 和 Com_rollback
可以了解事务提交和回滚的情况，对于回滚操作非常频繁的数据库，可能意味着应用编写存在问题。

>* connections:试图连接Mysql服务器的次数。

>* uptime:服务器工作的时间。

>* slow_queries:慢查询的次数。


慢查询日志记录了所有执行时间超过参数long-query_time(单位：秒)设置值的语句，long_query_time 默认为10秒，最小为0 可以通过 set long_query_time=? 来设置 ，慢查询日志文件存在指定路径下，默认文件名是host_name-slow.log.



## 2.通过EXPLAIN 分析低效的SQL执行计划



>* type：表示MySql在表中找到所需行的方式，或者叫访问类型：
 
   ALL-->index-->range-->ref-->eq_ref-->const,system-->NULL
 
   (1) type=ALL ,全表扫描，MySql遍历全表来找到匹配的行

   (2) type=index,索引全扫描，MySql遍历整个索引来查询配置的行
 
   (3) type=range, 索引范围扫描，常见于<,<=,>,>=,between等操作符
 
   (4) type=ref,使用非唯一索引扫描或唯一索引的前缀扫描，返回匹配某个单独值的记录行,ref还经常出现在join     操作中
 
   (5) type=eq_ref，类似于ref，区别就在使用的索引是唯一索引，对应每个索引键值，表中只有一条，记录匹配；   简单来说，就是多表连接中使用primary key或者 unique index作为关联条件
 
   (6)type=const/system,单表中最多有一个匹配行，查询起来非常迅速，所以这个匹配行中的其他列的值可以被优化   器在当前查询中当中常量来处理，例如，根据主键primary key或者唯一索引unique index 进行的查询
 
   (7) type=null，Mysql不用访问表或者索引，直接就能够得到结果


>* select_type

表示SELECT的类型，常见的取值有SIMPLE(简单表，即不使用表连接或者子查询),PRIMARY(主查询，即外层的查询)，UNION（UNION中的第二个或者后面的查询语句），SUBQUERY（子查询中的第一个SELECT）等。

>* possible_keys:表示查询时可能使用的索引

>* key:表示实际使用的索引

>* key_len :使用到索引字段的长度

>* rows:扫描行的数量

>* Extra:执行情况的说说明和描述，包含不适合在其他列中显示但是对执行计划非常重要的额外信息。

#### EXPLAIN EXTENDED:
 >* MySQL4.1开始引入了explain extended 命令，通过explain extended 加上show warnings我们能够看到在SQL真正被执行之前优化器做了哪些SQL改写
 
#### 通过trace 分析优化器如何选择执行计划
通过trace文件能够进一步了解为什么优化器选择A执行计划而不是B执行计划。
>* SET OPTIMIZER_TRANCE="enabled=on",END_MARKERS_IN_JSON=on;(1.打开trace，设置格式为Json)
>* SET OPTIMIZER_TRACE_MAX_MEN_SIZE=1000000;(2.设置trace最大能够使用的内存大小,避免解释过程中因为内存过小 而不能够完整的显示。)

>* 执行相关的语句。

>* select * FROM INFORMATION_SCHEMA.OPTIMIZER_TRACE; 分析跟踪文件 
 
 
 
#### EXPLAIN  PARTITIONS

>* MySQL 5.1开始支持分区功能，通过该命令查看SQL所访问的分区




## 3.通过Show Profile 分析SQL

 MySQL从5.0.37版本增加了对show profiles 和 show profile语句的支持。
 
 >* select @@have_profiling   (判断当前的SQL版本是否支持 profile)
 
 
 >* select @@profiling (查询profiling是否是开启的)
 
 >* SET  profiling=1  (设置开启或者关闭 1：开启 0：关闭)
 
 举例：select count(*) from SGPProductBaseInfo
 
 执行该语句之后： 执行 SHOW  profiles;
 
 可以看到：QUERY_ID，DURATION，QUERY 分别是执行SQL的所对应的ID，耗时，所对应的SQL
 
 执行完毕之后，找到你要分析的SQL找到其对应的Query_Id,比如为12；
 
 show profile for query 12; 能够看到改SQL执行过程中线程的每个状态和消耗的时间
 
 MySQL支持进一步选择 all，cpu，block io，context switch,page  faults等明细类型来查看MySQl在使用什么资源上耗费了过高的时间：
 
 例如：show profile cpu for query 12；
 
 
 
## 4.使用索引

| 索引        | MyISAM引擎   | innoDB 引擎 | Memory引擎  |
| --------   | -----:  | :----:  |
| B-Tree   | 支持 |   支持    |支持|
| HASH      |   不支持   |   不支持   |支持|
|R-Tree        |    支持    |  不支持 |不支持|
|Full-text       |    支持    |  暂不支持 |不支持|

索引是存储引擎用于快速查找记录的一种数据结构（B+Tree），通过合理的使用数据库索引可以大大提高系统的访问性能

> ##### 索引的类型及操作

 1.主键索引
   一种特殊的唯一索引，不允许有空值，一般在建表的同时创建主键索引
 
 ALTER TABLE 'table_name’ ADD PRIMARY KEY 'index_name' ('column');

  
 2.普通索引
   最基本的索引，没有任何限制

   ALTER TABLE 'table_name’ ADD INDEX 'index_name' ('column',...);
 
 3.唯一索引
  与普通索引类似，不同就是索引列的值必须唯一，可以为空值，如果是组合索引，列值的组合必须唯一

 ALTER TABLE 'table_name’ ADD UNIQUE 'index_name' ('column',...);

 4.全文索引
 
  MySQL支持全文索引和搜索功能。MySQL中的全文索引类型为FULLTEXT的索引。  FULLTEXT 索引仅可用于       MyISAM表；

ALTER TABLE 'table_name’ ADD FULLTEXT 'index_name' ('column',...);

 5.删除索引

  DROP INDEX 索引名称 ON 表名;
 
 6.如何查看索引

 SHOW INDEX FROM 表名;
 
 > ##### 复合索引遵循最左匹配原则
 >* Table:表的名称。
 >* Non_unique:如果索引不能包括重复词，则为0。如果可以，则为1。
 >* Key_name:索引的名称。
 >* Seq_in_index :索引中的列序列号，从1开始。
 >* Column_name :列名称。
 >* Collation:列以什么方式存储在索引中。在MySQL中，有值‘A’（升序）或NULL（无分类）。
 >* Cardinality :索引中唯一值的数目的估计值。通过运行ANALYZE TABLE或myisamchk -a可以更新。基数根据被存储为整数的统计数据来计数，所以即使对于小型表，该值也没有必要是精确的。基数越大，当进行联合时，MySQL使用该索引的机会就越大。
 >* Sub_part :如果列只是被部分地编入索引，则为被编入索引的字符的数目。如果整列被编入索引，则为NULL。
 >* Packed  :指示关键字如何被压缩。如果没有被压缩，则为NULL。
 >* Null  :如果列含有NULL，则含有YES。如果没有，则该列含有NO。
 >* Index_type   :用过的索引方法（BTREE, FULLTEXT, HASH, RTREE）。
 >* Comment    :多种评注。
 
 1.不按索引最左列开始查询时，则不会使用到索引 
 
 2.查询中某个列有范围查询时，则其右边的所有列都不会使用到索引

 3.查询中的某个列没有使用到索引时，则其右边的所有列都不会使用到索引 

> ##### 存在索引但是不能使用索引的典型场景

 (1)以%开头的like 查询不能够利用B-Tree索引。
 
 (2) 数据类型出现隐式转换的时候也不能使用索引，特别是当列类型是字符串，那么一定记得在where条件中把字     符串常量值用引号引起来，否则即便是这个列上有索引，MySql也不会用到。
 
 (3)复合索引的情况下，假如查询条件不含包索引列最左边的部分。
 
 (4)如果MySql估计使用索引比全表扫描更慢，则不能使用索引
 
 (5)用or分隔开的条件，如果OR前的条件中的列有索引，而后面的列中没有索引，那么涉及到的索引都不会被用到
 
 
 > ##### 查看索引的使用情况（show status like 'Handler_read%'）
 
  1.Handler_read_key的值很高表示索引使用率越高
  
  2.Handler_read_rnd_next的值高意味着查询运行低效，并且应该建立索引补救。
  
  
## 5.常用的Sql优化

  >优化 Order By
  
   尽量减少额外的排序，通过索引直接返回有序数据。where条件和order by 使用相同的索引，并且order by 的顺序和索引顺序相同。并且order by 的字段都是升序或者降序。否则肯定需要额外的排序操作，这样就会出现FileSort。
   
   总结，下列SQL可以使用索引：
   select * FROM tabname order by key_part1,key_part2....;
   
   slect * FROM tabname where key_part1=1 order by key_part1 desc,key_part2 desc;
   
   select * FROM tabname order by key_part1 desc,key_part2 desc;
   
   但是在以下几种情况下则不使用索引：
   
   select * from tabname order by key_part1 desc,key_part2 asc,...;
    -- order by 的字段包含asc和desc
    
   select * from tabname where key2=constant order by key1;
   --用于查询行的关键字与order by中所使用的不同
   
   select * from tabname order by key1,key2;
   --对不同的关键字使用order by;
   
   >优化Group by 语句
   
   如果查询包括 GROUP BY 但用户想要避免排序结果的消耗，则可以指定ORDER BY NULL 禁止排序、
   
  >嵌套查询
  
  使用子查询可以一次地完成很多逻辑上需要多个步骤才能完成的SQL操作，同时也可以避免事务或者表锁死，并   且写起来也很容易。但是，有些情况下，子查询可以更有效率的连接（JOIN）替代。
   
   
   
  >优化OR条件
  
  对于含有OR的查询子句，如果要利用索引，则OR之间的每个条件都必须用到索引； 
   
 >优化分页查询
 
    1.第一种优化思路
 
 在索引上完成排序分页的操作，最后根据主键关联回表查询所需要的其他列的内容。
 
   比如：优化前  select film_id,desc from film order by title limit 50,5;
   
   优化后： select a.film_id,a.desc from film a INNER JOIN (select film_id from film order by title limit 50,5) b on a.film_id=b.film_id
   
   可以从查看执行计划结果中看到优化后不再是全表扫描了
   
     2.第二种优化思路：
把limit查询转换成某个位置的查询，例如，假设每页10条记录，查询表42页的记录并按照主键倒序排序

  select * from table order by id desc limit 410,10;
  
  此时是全表扫描的
  
  优化： 可以在分页的时候传递一个参数，此参数就是第41页最后一条记录的Id，例如为12121，
  
  就可以改写为 select * from table where id<12121 order by id desc limit 10;
   
## 6.使用SQL提示

  1.USE INDEX
  在查询语句中表名的后面，添加USE INDEX 来提供希望MySQl去参考的索引列表，就可以让MySql 不再考虑其他可用的索引。
  
  explain select * from table use index (idx_'column1'_'column2')
  
  
  2.IGNORE INDEX
  
  如果MySQl 忽略一个或者多个索引，则可以使用IGNORE INDEX 与USE INDEX 相反
  
  
  3.FORCE INDEX
  
  为了强制Mysql 使用一个特定的索引，可以使用FORCE INDEX，而use index    只是让执行计划优先去参考索引，并没有强制的意思
  

 
### 7.三个简单使用的优化方法

>* 定期优化表 optimize table tablename1，tablename2...

.如果已经删除了表的一大部分，或者如果已经对含有可变长度行的表（含有varchar，BLOB或 TEXT 列的表）进行了很多的更改，这个命令可以将表中的空间碎片进行合并，并且可以消除由于删除或者更新造成的空间浪费。但是此命令 只对 MyISAM，BDB，InnoDB 表起作用。


>* analyze table tablename;

本语句用于分析和存储表的关键字分布，分析的结果将可以使得系统得到准确的统计信息，使得SQL 能够生成正确的执行计划。如果用户感觉实际执行计划并不是预期的执行计划，执行一次分析表可能会解决问题。只对 MyISAM，BDB，InnoDB 表起作用。


> * check table tablename
  
  
检查表的作用是检查一个或者多个表是否有错误，同样也可以检查视图是否有错误。比如：

创建一个视图 ： create view viewname as select * from tablename;

check 一下该视图  check table viewname; 此时正确

删掉视图依赖的表： drop table tablename；

此时再check一下刚才的 视图，发现已出错。




  
 
  



 

 
 
 
