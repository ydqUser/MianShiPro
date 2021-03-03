
<!-- TOC -->

- [base](#base)
    - [范式](#范式)
    - [DDL DML](#ddl-dml)
    - [复合主键与联合主键](#复合主键与联合主键)
- [配置](#配置)
    - [information_schema](#information_schema)
- [逻辑架构](#逻辑架构)
    - [表空间](#表空间)
    - [存储架构](#存储架构)
        - [段(segment)](#段segment)
        - [区(extent)](#区extent)
        - [页(page)](#页page)
    - [表优化](#表优化)
- [存储引擎](#存储引擎)
    - [InnoDB](#innodb)
        - [四大特性](#四大特性)
    - [MyISAM与InnoDB的区别](#myisam与innodb的区别)
- [查询](#查询)
- [索引](#索引)
    - [全文索引](#全文索引)
- [事务](#事务)
    - [事务模型ACID](#事务模型acid)
    - [隔离级别](#隔离级别)
        - [一致性保证](#一致性保证)
- [锁](#锁)
    - [MVCC](#mvcc)
    - [使用方式：乐观锁、悲观锁](#使用方式乐观锁悲观锁)
    - [锁级别：共享锁、排他锁、意向锁、间隙锁](#锁级别共享锁排他锁意向锁间隙锁)
    - [锁粒度：行级锁、表级锁、页级锁](#锁粒度行级锁表级锁页级锁)
    - [按操作&加锁方式](#按操作加锁方式)
        - [操作：DDL锁、DML锁](#操作ddl锁dml锁)
        - [加锁方式：自动锁、显示锁](#加锁方式自动锁显示锁)
    - [死锁](#死锁)
- [主从同步](#主从同步)
- [分区分库分表](#分区分库分表)
    - [分区](#分区)
    - [分表](#分表)
    - [分库](#分库)
- [8.0和5.7的区别](#80和57的区别)
    - [隐藏索引](#隐藏索引)
    - [设置持久化](#设置持久化)
    - [UTF-8编码](#utf-8编码)
    - [通用表达式(Common Table Expressions)](#通用表达式common-table-expressions)
    - [窗口函数](#窗口函数)
- [小点](#小点)
    - [drop truncate delete区别](#drop-truncate-delete区别)
- [错误解决办法](#错误解决办法)
    - [根据日志数据找回](#根据日志数据找回)
    - [blocked because of many connection errors](#blocked-because-of-many-connection-errors)
- [other](#other)
    - [事务日志](#事务日志)
        - [redo log](#redo-log)
        - [undo log](#undo-log)
        - [binlog和事务日志的先后顺序及group commit](#binlog和事务日志的先后顺序及group-commit)

<!-- /TOC -->



[refer(整体)](https://juejin.cn/post/6883270227078070286#heading-65)
[官方文档](https://www.mysqlzh.com/doc/215/418.html)

list：
- [ ] [知乎mysql底层原理及性能优化](https://zhuanlan.zhihu.com/p/161233931)
- [ ] [知乎mysql底层原理及性能优化2](https://zhuanlan.zhihu.com/p/157698912)


### base

#### 范式
- 第一范式(1NF)：属性不可分
- 第二范式(2NF)：每个非主属性完全依赖于键码
- 第三范式(3NF)：非主属性不传递依赖于键码

#### DDL DML
DDL （Data Definition Language 数据定义语言）
     create table 创建表    
     alter table  修改表   
     drop table 删除表   
     truncate table 删除表中所有行    
     create index 创建索引   
     drop index  删除索引 
     当执行DDL语句时，在每一条语句前后，oracle都将提交当前的事务。如果用户使用insert命令将记录插入到数据库后，执行了一条DDL语句(如create table)，此时来自insert命令的数据将被提交到数据库。当DDL语句执行完成时，DDL语句会被自动提交，不能回滚。 
DML （Data Manipulation Language 数据操作语言）
     insert 将记录插入到数据库 
     update 修改数据库的记录 
     delete 删除数据库的记录 
     当执行DML命令如果没有提交，将不会被其他会话看到。除非在DML命令之后执行了DDL命令或DCL命令，或用户退出会话，或终止实例，此时系统会自动发出commit命令，使未提交的DML命令提交。
#### 复合主键与联合主键
- 复合主键：表的主键含有一个或以上的字段组成
- 联合主键：多个主键联合形成一个主键的组合。在多对多模型中，需要两个表中的主键组成联合主键，来确定一条记录

联合主键(感觉应该是复合主键)自增：(要区分数据库引擎)
- 使用mysql联合主键自增，需要使用MYISAM作为存储引擎。对于AUTO_INCREMENT类型的字段，InnoDB中必须包含只有该字段的索引
- 使用联合主键自增的时候，自增键不能是主键最左的键。
    - 当联合主键第一列自增时，只会自己自增，不会根据第二列的值恢复到1
    - 当联合主键第二列自增时，会根据第一列的值的变化将值恢复到1

<details>
  <summary>联合主键自增示例</summary>
  <pre><code> 

  -- myIsam引擎
     drop table test_isam;
create table test_isam(
	vid int(3) unsigned not null,
    id int(10) unsigned not null auto_increment,
    name varchar(10) character set utf8,
    age int(10),
    primary key(vid,id)
   ) engine=MyISAM;

insert into test_isam (vid, name, age) values (1, 'a', 2), (2, 'b', 2), (2, 'c', 3), (2, 'd', 4), (3, 'e', 5), (2, 'f', 6);
select * from test_isam; # 可以查看发现 id则会根据vid的值的变化，而恢复到1

-- innodb引擎
create table test_pri(
	vid int(3) unsigned not null,
    id int(10) unsigned not null auto_increment,
    name varchar(10) character set utf8,
    age int(10),
    primary key(vid,id)
   );  # Error : Incorrect table definition; there can be only one auto column and it must be defined as a key
  </code></pre>
</details>

#### Union和Union All

- Union：对两个结果集进行并集操作，不包括重复行，同时进行默认规则的排序；
- Union All：对两个结果集进行并集操作，包括重复行，不进行排序

### 配置

- 查看数据存放位置 `show variables like 'datadir';`   # /var/lib/mysql
- 查看版本 
    - select version();
    - 命令行 `> status` 还可以查看连接数等信息


#### information_schema
information_schema.tables字段说明
- `row_format` 行格式
    - `fixed` 若一张表里不存在varchar/text及其变形/blob及其变形的字段的话，这个表也叫静态表，即该表的`row_format`是fixed，就是说每条记录所占用的字节一样。优点读取快，缺点浪费一部分空间
    - `dynamic` 若一张表里存在varchar/text及其变形/blob及其变形的字段的话，这个表也叫动态表，即该表的`row_format`是dynamic，就是说每条记录所占用的字节是动态的。优点节省空间，缺点增加读取的时间开销
    > 所以做搜索查询量大的表一般都以空间来换取时间，设计成静态表
    - 其他值：`DEFAULT | FIXED | DYNAMIC | COMPRESSED | REDUNDANT | COMPACT`
    > 修改行格式： `ALTER TABLE table_name ROW_FORMAT = DEFAULT`
- `index_length` 索引长度
- `data_free` 空间碎片
    - 查询数据库空间碎片 `select table_name,data_free,engine from information_schema.tables where table_schema='yourdatabase';`
    - 对数据表优化 `optimize table table_name;`

### 逻辑架构

[refer:段 区 页](https://juejin.cn/post/6919263740122628109)
[refer:分区](https://www.cnblogs.com/helios-fz/p/13671682.html)
#### 表空间
- 系统表空间：存储MySQL内部的数据字典数据，如information_schema下的数据
- 用户表空间：当开启innodb_file_per_table=1时，数据表从系统表空间独立出来存储在以table_name.ibd命令的数据文件中，结构信息存储在table_name.frm文件中
- undo表空间：存储Undo信息，如快照一致读和flashback都是利用undo信息
- 从MySQL 8.0开始允许用户自定义表空间 ``

mysql8.0

#### 存储架构
##### 段 segment
- 数据段
- 索引段
- 回滚段

在InnoDB存储引擎中，【对段的管理是由引擎自身所完成的？？】
##### 区 extent
区是由连续的页组成的空间，无论页的大小怎么变，区的大小默认总是为1MB。为了保证区中的页的连续性，InnoDB存储引擎一次从磁盘申请4-5个区，InnoDB页的大小默认为16kb，即一个区一共有64（1MB/16kb=16）个连续的页。每个段开始，先用32页（page）大小的碎片页来存放数据，在使用完这些页之后才是64个连续页的申请。这样做的目的是，对于一些小表或者是undo类的段，可以开始申请较小的空间，节约磁盘开销


##### 页 page

页是InnoDB磁盘管理的最小单位。默认大小为16KB，可以通过参数innodb_page_size来设置。常见的页类型有：数据页，undo页，系统页，事务数据页，插入缓冲位图页，插入缓冲空闲列表页，未压缩的二进制大对象页，压缩的二进制大对象页等

任何一个页都由页头、页身和页尾组成

页头
页身
    - 表空间首页头信息区
    - 业务数据区


- FSP HDR页 【一个表空间可能对应多个数据文件】
    - 0号文件的0号page页中存储表空间信息以及当前表空间所拥有的段链表的指针
- XDES页
- INODE页


#### 表优化
[refer](https://blog.csdn.net/e421083458/article/details/19907097)

整理表碎片化空间方法:
- alter table tabname engine=InnoDB; # 
- ANALYZE table tabname;
- optimize table tabname;

- Analyze Table tabname (修复索引/ 优化表的统计信息)
    - 语法：ANALYZE [NO_WRITE_TO_BINLOG|LOCAL] TABLE tabname
        - 默认的，MySQL服务会将 ANALYZE TABLE语句写到binlog中，以便在主从架构中，从服务能够同步数据。（从服务通过binlog与主服务完成数据同步）。可以添加参数取消将语句写到binlog中
    - MySQL 的Optimizer(优化元件)在优化SQL语句时，首先需要收集一些相关信息，其中就包括表的cardinality（可以翻译为“散列程度”），它表示某个索引对应的列包含多少个不同的值
        - `show index from tabname` 查看cardinality(索引基数)。如果值太低，说明索引基本失效；如果值太高，说明可能是记录有较多的删除导致的碎片，从而引起统计信息不准确
    - 限制：执行此语句需要具有SELECT、DELETE权限，且只对存储引擎为InnoDB、MyISAM、NDB的表有作用，不能用于视图
    - 执行时，会对表加上读锁
    - 生产操作的风险：
        - analyze table的需要扫描的page代价粗略估算公式：sample_pages * 索引数 * 表分区数。
        - 因此，索引数量较多，或者表分区数量较多时，执行analyze table可能会比较费时，要自己评估代价，并默认只在负载低谷时执行。
        - 特别提醒，如果某个表上当前有慢SQL，此时该表又执行analyze table，则该表后续的查询均会处于waiting for table flush的状态，严重的话会影响业务，因此执行前必须先检查有无慢查询
- optimize table tabname (对表进行碎片整理)
    - `show table status like 'table_name'` # 可以看到数据大小 碎片大小
    - 重新组织表数据和关联索引数据的物理存储，以减少存储空间并提高访问表时的I/O效率。通俗点理解就是碎片整理。它会重建表以更新索引统计信息
    - Optimize Table语句对MyISAM和InnoDB类型的表都有效
    - Optimize Table也可以使用local来取消写入binlog
    - 当删除数据时，mysql并不会回收被已删除的数据占用的存储空间，以及索引位。而是空在那里，等待新的数据来填补这个空缺

优化示例(分区)：
1. 建表
```mysql
CREATE TABLE `withdraw_range` (
  `id` int(11) NOT NULL AUTO_INCREMENT
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=COMPACT COMMENT='流水'
PARTITION BY RANGE (`month`) (
    PARTITION p1 VALUES LESS THAN (2),
    PARTITION p12 VALUES LESS THAN (13)
);

# 这样会创建多个分区数据文件
```
2. 原来数据备份或者入新表
3. 清理批量删除数据导致的磁盘碎片
```mysql
select table_name, table_rows, data_length, index_length, data_free from information_schema.tables;
SELECT DATA_FREE FROM information_schema.tables WHERE TABLE_NAME = 'extension_withdraw_range' and TABLE_SCHEMA = 'db_name';
alter table table_name ENGINE=InnoDB 
```

> 清理磁盘碎片会导致锁表，谨慎使用
### 存储引擎

#### InnoDB

##### 四大特性
[refer](https://www.cnblogs.com/zhs0/p/10528520.html)
- 插入缓冲(insert buffer)
    - 插入缓冲（Insert Buffer/Change Buffer）：提升插入性能，change buffering是insert buffer的加强，insert buffer只针对insert有效，change buffering对insert、delete、update(delete+insert)、purge都有效
    - 只对于非聚集索引（非唯一）的插入和更新有效，对于每一次的插入不是写到索引页中，而是先判断插入的非聚集索引页是否在缓冲池中，如果在则直接插入；若不在，则先放到Insert Buffer 中，再按照一定的频率进行合并操作，再写回disk。这样通常能将多个插入合并到一个操作中，目的还是为了减少随机IO带来性能损耗。
    - 使用插入缓冲的条件：
        - 非聚集索引。聚集索引一般是自增的，写入的位置是顺序的；非聚集索引等于随机写，效率较低
        - 非唯一索引。在插入缓冲时，数据库并不会去查找索引页来判断插入的记录是否唯一，如果去查找肯定又会发生离散读取的情况，那就导致插入缓冲失去了意义，因此唯一索引列不能使用插入缓冲
    
    - 将insert buffer进行合并到真正的索引页的情况发生在如下：
        - 非唯一聚集索引页被读取到缓冲池时：执行select查询时，会检查非唯一聚集索引是否有记录存放在Insert Buffer中，如果有，则对该页的所有记录合并插入到实际索引列中
        - 非唯一聚集索引页无可用空间时：检测到页的空间小于1/32时，会强制进行一次合并插入操作。
        - Master Thread执行：在后台主线程中，每秒或每10秒就会进行一次合并插入缓冲的操作
- 二次写(double write)：double write带给InnoDB存储引擎的是数据页的可靠性。
    - double write由两部分组成，一部分是内存中的doublewrite buffer，另一部分是物理磁盘上共享表空间中连续128个页，即2个区。两者大小都为2MB。
    - 在对缓存池的脏页进行刷新的时候，并不是直接写入到磁盘，而是先将脏页复制到内存中的doublewrite buffer中，之后通过doublewrite buffer分两次，
      每次1MB顺序的写入共享表空间的物理磁盘上，(因为这个过程是顺序写的，开销并不是很大)；再将doublewrite buffer 中的页写入各个 表空间文件中，此时的写入则是离散的。
- 自适应哈希索引(ahi Adaptive Hash index)
    - 当某个非聚集索引被等值查询的次数很多时，就会为这个非聚集索引再构造一个hash索引，这个hash索引会放在缓存中
- 预读(read ahead)：线性预读放到以extent为单位，而随机预读放到以extent中的page为单位。线性预读着眼于将下一个extent提前读取到buffer pool中，而随机预读着眼于将当前extent中的剩余的page提前读取到buffer pool中
    - 线性预读：如果一个extent中的被顺序读取的page超过或者等于`配置参数innodb_read_ahead_threshold`时，Innodb将会异步的将下一个extent读取到buffer pool中
    - 随机预读：当同一个extent中的一些page在buffer pool中发现时，Innodb会将该extent中的剩余page一并读到buffer pool中，由于随机预读方式给Innodb code带来了一些不必要的复杂性，同时在性能也存在不稳定性，在5.5中已经将这种预读方式废弃

#### MyISAM与InnoDB的区别
[refer](https://blog.csdn.net/qq_35642036/article/details/82820178)
- 事务：InnoDB支持事务，MyISAM不支持
- 外键：InnoDB支持外键，而MyISAM不支持。对一个包含外键的InnoDB表转为MYISAM会失败
- 是否聚集索引：
    - InnoDB是聚集索引，使用B+Tree作为索引结构，数据文件是和（主键）索引绑在一起的（表数据文件本身就是按B+Tree组织的一个索引结构），必须要有主键，通过主键索引效率很高。但是辅助索引需要两次查询，先查询到主键，然后再通过主键查询到数据。因此，主键不应该过大，因为主键太大，其他索引也都会很大
    - MyISAM是非聚集索引，也是使用B+Tree作为索引结构，索引和数据文件是分离的，索引保存的是数据文件的指针。主键索引和辅助索引是独立的
    - 即：InnoDB的B+树主键索引的叶子节点就是数据文件，辅助索引的叶子节点是主键的值；而MyISAM的B+树主键索引和辅助索引的叶子节点都是数据文件的地址指针
- 行锁：InnoDB支持表、行(默认)级锁，而MyISAM支持表级锁
    - InnoDB的行锁是实现在索引上的，而不是锁在物理行记录上。潜台词是，如果访问没有命中索引，也无法使用行锁，将要退化为表锁
- 唯一索引：InnoDB表必须有唯一索引（如主键）（用户没有指定的话会自己找/生产一个隐藏列Row_id来充当默认主键），而Myisam可以没有
- InnoDB不保存表的具体行数，执行`select count(*) from table`时需要全表扫描。而MyISAM用一个变量保存了整个表的行数，查看行数时只需要读出变量即可
    - InnoDB没有这个变量主要因为事务特性，在同一时刻表中的行数对于不同的事务而言是不一样的，因此count统计会计算对于当前事务而言可以统计到的行数，而不是将总行数储存起来方便快速查询。InnoDB会尝试遍历一个尽可能小的索引除非优化器提示使用别的索引。如果二级索引不存在，InnoDB还会尝试去遍历其他聚簇索引
    - 如果得到大致的行数值够满足需求可以用`SHOW TABLE STATUS`
- Innodb存储文件有frm、ibd，而Myisam是frm、MYD、MYI
    - Innodb：frm是表定义文件，ibd是数据文件
    - Myisam：frm是表定义文件，myd是数据文件，myi是索引文件
    
    

### 查询

### 索引

- hash：不支持范围查询
- B+tree：支持范围查询
myisam和innodb都支持b+tree和hash。
索引提高查询效率，但是牺牲了插入和删除的性能，要维护索引的结构。对于唯一性较差、频繁更改的字段不应该建立索引，唯一性较高、经常筛选、排序的字段
索引失效：or、is null、!=

#### 全文索引

[refer](https://blog.csdn.net/mrzhouxiaofei/article/details/79940958)

fulltext，通过关键字的匹配来进行查询过滤，需要基于相似度的查询，而不是精确数值比较，这时需要全文索引。比 like %%快n倍

支持情况：
- MySQL 5.6 以前的版本，只有 MyISAM 存储引擎支持；MySQL 5.6 及以后的版本，MyISAM 和 InnoDB 存储引擎均支持。
- 只有字段的数据类型为 char、varchar、text 及其系列才可以建全文索引

创建索引：`alter table fulltext_test add fulltext index content_tag_fulltext(content,tag);`
使用：全文索引使用match和against关键字
    - `select * from fulltext_test where match(content,tag) against('xxx xxx')`
    - match() 函数中指定的列必须和全文索引中指定的列完全相同，否则就会报错，无法使用全文索引，这是因为全文索引不会记录关键字来自哪一列。如果想要对某一列使用全文索引，请单独为该列创建全文索引

两种全文索引：
- 自热语言的全文索引。默认情况或者`in natural language mode 修饰符`,match() 函数对文本集合执行自然语言搜索
- 布尔全文索引。在查询中自定义某个被搜索的词语的相关性。eg：`select * test where match(content) against('a*' in boolean mode);` 会查询到a/aa/aaa
    - `+`必须包含该词
    - `-`必须不包含该词
    - `>`提高该词的相关性，查询的结果靠前
    - `<`降低该词的相关性，查询的结果靠后
    - `*`通配符，只能接在词后面

### 事务

#### 事务模型ACID

- A : atomicity 原子性。原子性是我们对事务最直观的理解：事务就是一系列的操作，要么全部都执行，要么全部都不执行。`通过UndoLog来实现`
- C : consistency 一致性。数据库事务不能破坏关系数据的完整性以及业务逻辑上的一致性。
- I : isolation 隔离性。在并发环境中，当不同的事务同时操纵相同的数据时，每个事务都有各自的完整数据空间。`通过锁机制实现`
- D : durability 持久性。只要事务成功结束，它对数据库所做的更新就必须永久保存下来。即使发生系统崩溃，重新启动数据库系统后，数据库还能恢复到事务成功结束时的状态`通过RedoLog来实现`

原子性是事务隔离的基础，隔离性和持久性是手段，最终目的是为了保持数据的一致性

#### 隔离级别

- 读未提交(READ UNCOMMITTED)：导致脏读、不可重复读、幻读
- 读已提交(READ COMMITTED)：导致不可重复读、幻读
- 可重复读(REPEATABLE READ)(默认的隔离级别)：**保证同一事务中多次读取相同数据的结果是一样的**。导致幻读
- 串行化(SERIALIZABLE)

##### 一致性保证
[refer](https://blog.csdn.net/weixin_39781452/article/details/110864632)
[refer](https://mp.weixin.qq.com/s/qHzb6oPrrbAPoIlfLJVNAg)

InnoDB多版本的实现：
1. 三个隐藏字段：
- DB_TRX_ID字段，6字节。表示插入或更新行的最后一个事务的事务标识符。此外，删除在内部被视为更新，其中行中的特殊位被设置为将其标记为已删除
- DB_ROLL_PTR字段，7字节，叫做回滚指针（roll pointer）。回滚指针指向写入回滚段的撤消日志（Undo Log）。如果行已更新，则撤消日志包含重建更新前该行内容所需的信息
- DB_ROW_ID字段，6字节。包含一个随着新行插入而单调增加的行ID，如果innodb自动生成聚集索引，则该索引包含行ID值。否则，DB_ROW_ID列不会出现在任何索引中
2. 多版本的产生过程：
- 插入一条记录时，记录对应的回滚指针为null
- 更新记录时，实际完成的事情：
    - 用排它锁锁定该行
    - 把该行修改前的值拷贝到Undo Log中
    - 修改当前行的值，填写事务编号，使回滚指针指向Undo Log中的修改前的行
    - 记录Redo Log，包括Undo Log中的变化
多次更新后，回滚指针会把不同版本的记录串在一起。在InnoDB中存在purge线程，它会查询那些比现在最老的活动事务还早的Undo Log，并删除它们，从而保证Undo Log文件不至于无限增长
3. 提交与回滚
- 事务正常提交时，InnoDB只需要更改事务状态为commit即可，不需要做其他额外的工作
- 事务回滚时，需要根据当前回滚指针从Undo Log中找到事务修改前的版本，并恢复。如果事务影响的行非常多，回滚则可能会很慢，根据经验值没提交的事务行数在1000~10000之间，InnoDB效率还是非常高的

### 锁

#### MVCC
多版本并发控制(Multi-Version Concurrency Control, MVCC)

基础概念
- 版本号
    - 系统版本号：递增
    - 事务版本号：
- 隐藏的列：MVCC每行记录后面都保存着两个隐藏的列，用来存储两个版本号：
    - 创建版本号：指示创建一个数据行的快照时的系统版本号
    - 删除版本号：
- Undo日志

#### 使用方式：乐观锁、悲观锁

- 悲观锁：select * from tb_table where id = 1 for update;  # 要确定走了索引，而不是全表扫描，否则会将整个数据表锁住

悲观锁并不适应于任何场景，因为悲观锁大多数情况下依靠数据库的锁机制实现，以保证操作最大程度的独占性。
如果加锁时间过长，其他用户长时间无法访问，影响程序的并发访问性，同时这样对数据库性能开销影响也很大，特别是对长事务而言，这时就需要乐观锁

#### 锁级别：共享锁、排他锁、意向锁、间隙锁

#### 锁粒度：行级锁、表级锁、页级锁

#### 按操作&加锁方式

##### 操作：DDL锁、DML锁

##### 加锁方式：自动锁、显示锁

#### 死锁

### 主从同步

主从复制：主从复制是指将主数据库的DDL和DML操作通过二进制日志传到从数据库上，然后在从数据库上对这些日志进行重新执行，从而使从数据库和主数据库的数据保持一致

原理：
- MySql主库在事务提交时会把数据变更作为事件记录在二进制日志Binlog中；
- 主库推送二进制日志文件Binlog中的事件到从库的中继日志Relay Log中，之后从库根据中继日志重做数据变更操作，通过逻辑复制来达到主库和从库的数据一致性；
- MySql通过三个线程来完成主从库间的数据复制，其中Binlog Dump线程跑在主库上，I/O线程和SQL线程跑着从库上；
- 当在从库上启动复制时，首先创建I/O线程连接主库，主库随后创建Binlog Dump线程读取数据库事件并发送给I/O线程，I/O线程获取到事件数据后更新到从库的中继日志Relay Log中去，之后从库上的SQL线程读取中继日志Relay Log中更新的数据库事件并应用，如下图所示

### 分区分库分表

#### 分区
[refer](https://www.cnblogs.com/helios-fz/p/13671682.html)

**概述**

将一个表中不同行的记录分配到不同的物理文件中，几个分区就有几个.idb文件(不是上面的extent)

MySQL数据库的分区是`局部分区索引`，一个分区中既存了数据，又放了索引。也就是说，每个区的聚集索引和非聚集索引都放在各自区的（不同的物理文件）。目前MySQL数据库还不支持全局分区。

**无论哪种类型的分区，如果表中存在主键或唯一索引时，分区列必须是唯一索引的一个组成部分**


**分区类型**
最常用的为RANGE分区。当插入的数据不在任意分区列中定义的时候，会抛出异常

- RANGE分区

行数据基于属于一个给定的连续区间的列值被放入分区。 YEAR()，TO_DAYS()，TO_SECONDS()，UNIX_TIMESTAMP()

- LIST分区

分区列的值是离散的，不是连续的。list分区使用values in，因为每个分区的值是离散的，因此只能定义值

- HASH分区

将数据均匀的分布到预先定义的各个分区中，保证每个分区的数量大致相同。增加分区后，需要重新计算
- 线性hash分区
增加分区后，还在原来的分区。相对于hash分区，没有那么均匀

- KEY分区

KEY分区和HASH分区相似，不同之处在于HASH分区使用用户定义的函数进行分区，KEY分区使用数据库提供的函数进行分区

**分区和性能**
查询分区是否起作用：explain partitions select * from tb_table; # partitions标识走了哪几个分区

- 数据库应用分为2类，一类是OLTP（在线事务处理），一类是OLAP（在线分析处理）
    - 对于OLAP应用分区的确可以很好的提高查询性能，因为一般分析都需要返回大量的数据，如果按时间分区，比如一个月用户行为等数据，则只需扫描响应的分区即可
#### 分表
- 水平分表：分表策略通常是用户id取模
    - 遇到的问题：
        - 跨表直接连接查询无法进行
        - 需要统计数据的时候
        - 如果数据持续增长，达到现有分表的瓶颈，需要增加分表，此时会出现数据重新排列的情况
    - 解决方案建议：
        - 增加汇总的冗余表。虽然数据量大，但是可以用于后台统计或查询时效性比较低的情况，而且可以提前算好某个时间点或时间段的数据
        - 分表瓶颈问题：最开始计算增长率，确定数据总量，提前计算需要的分表个数：考虑表分区，逻辑上还是一个表，实际物理存储在不同的物理地址上；分库
- 垂直分表：
    - 垂直拆分原则
        - 把大字段独立存储到一张表中
        - (使用频率)把不常用的字段单独拿出来存储到一张表
        - (使用用途)把经常在一起使用的字段可以拿出来单独存储到一张表
    - 垂直拆分标准
        - 表的体积大于2G并且行数大于1千万
        - 表中包含有text，blob，varchar(1000)以上
        - 数据有时效性的，可以单独拿出来归档处理
        
- 分表和分区的区别
    - 实现方式上：分表后每一个表都是完整的表，都对应三个文件（MyISAM引擎：一个.MYD数据文件，.MYI索引文件，.frm表结构文件）
    - 数据处理上：分表后数据存放在分表里，总表只是一个外壳，存取数据发生在一个一个的分表里面。分区只不过把存放数据的文件分成了许多小块，分区后的表还是一张表，数据处理还是由自己来完成
    - 提高性能上：分表后，单表的并发能力提高了，磁盘I/O性能也提高了。分区突破了磁盘I/O瓶颈，想提高磁盘的读写能力，来增加mysql性能
    - 实现难易程度：分表
    
    
实现方式上

mysql的分表是真正的分表，一张表分成很多表后，每一个小表都是完正的一张表，都对应三个文件（MyISAM引擎：一个.MYD数据文件，.MYI索引文件，.frm表结构文件）。

2，数据处理上

分表后数据都是存放在分表里，总表只是一个外壳，存取数据发生在一个一个的分表里面。分区则不存在分表的概念，分区只不过把存放数据的文件分成了许多小块，分区后的表还是一张表，数据处理还是由自己来完成。

3，提高性能上

分表后，单表的并发能力提高了，磁盘I/O性能也提高了。分区突破了磁盘I/O瓶颈，想提高磁盘的读写能力，来增加mysql性能。

在这一点上，分区和分表的测重点不同，分表重点是存取数据时，如何提高mysql并发能力上；而分区呢，如何突破磁盘的读写能力，从而达到提高mysql性能的目的。

4，实现的难易度上

分表的方法有很多，用merge来分表，是最简单的一种方式。这种方式和分区难易度差不多，并且对程序代码来说可以做到透明的。如果是用其他分表方式就比分区麻烦了。分区实现是比较简单的，建立分区表，跟建平常的表没什么区别，并且对代码端来说是透明的


#### 分库

- 分库方式
    - 可以通过取模的方式进行路由
    - 可以按照业务分库，`要注意处理好跨库事务`
- 作用：分表能解决单表数据量过大带来的查询效率下降，却无法给数据库的并发处理能力带来质的提升
    - 对数据库进行拆分，提高数据库写入能力(读可以用主从，从数据库读取)
    - 数据库连接数提高

### 8.0和5.7的区别

#### 隐藏索引
索引可以隐藏。隐藏索引对性能调试非常重要，索引可以被隐藏和显示。当一个索引隐藏时，不会被查询优化器所使用。
隐藏一个索引，然后观察数据库性能是否下降，如果下降，说明该索引有效，否则无效，可以删除

`ALTER TABLE t ALTER INDEX i INVISIBLE`
`ALTER TABLE t ALTER INDEX i VISIBLE`

#### 设置持久化

mysql的设置可以在运行时通过`SET GLOBAL`命令修改，但是只会临时生效，下次启动时数据库又会从配置文件中读取
mysql8新增了`SET PERSIST`命令，例如`Set persist max_connections = 500;` mysql会将该命令的配置保存在数据目录下的mysqld-auto.cnf文件中，下次启动时会读取该文件，用其中的配置来覆盖缺省的配置文件
#### UTF-8编码

从mysql 8 开始，数据库的编码改为utf8mb4

#### 通用表达式(Common Table Expressions)

可以使嵌套式表的查询层次更加分明：
```
SELECT t1.*, t2.* FROM
	 (SELECT col1 FROM table1) t1,
	 (SELECT col2 FROM table2) t2;

WITH
	 t1 AS (SELECT col1 FROM table1),
	 t2 AS (SELECT col2 FROM table2)
	SELECT t1.*, t2.* 
	FROM t1, t2;
```
#### 窗口函数

```
select * from classes;  // 例如结果为：class1 41; class2 43; class3 54

如果对班级人数从小到大排序：
select *, rank() over w as `rank` from classes window w as (order by stu_count); // 结果为：class1 41 1; class2 43 2; class3 54 3
```

### 小点

#### drop truncate delete区别
- 速度上 drop > truncate > delete
- delete执行删除的过程是每次从表中删除一行，并且同时将该行的删除操作作为事务记录在日志中保存以便进行回滚操作。TRUNCATE TABLE 则一次性地从表中删除所有的数据并不把单独的删除操作记录记入日志保存，删除行是不能恢复的
- 表和索引所占空间
    - 当表被TRUNCATE 后，这个表和索引所占用的空间会恢复到初始大小，
    - DELETE操作不会减少表或索引所占用的空间。
    - drop语句将表所占用的空间全释放掉。

### 错误解决办法

#### 根据日志数据找回

```
> CREATE TABLE newTableName AS (SELECT * FROM sourceTableName  where whereExpression); # 备份表

> show master status;  # 查看状态 查看最后一个binlog日志的编号名称 及其最后一个操作事件的pos结束点
> show master logs; # 查看日志列表
> show variables like 'log_%';  # 可以查看日志是否开启log_bin log目录

恢复：
mysqlbinlog master-bin.000014|grep -5a "DROP TABLE"  # 找到删除命令的日志点  有一个 # at 3232 这个数字为drop table操作的事件点
mysqlbinlog master-bin.000014 -d t1  --skip-gtids --stop-position=2439 > test.sql # -d参数指定数据库 转成sql 然后执行sql即可

mysqlbinlog --no-defaults mysql-bin.000001 -v > 11.txt 可以看到sql执行内容 进而查看
mysqlbinlog --no-defaults mysql-bin.000001 -v | grep "INSERT INTO \`zdassistant_preprod20200730before\`.\`sys_config\`" -A 10

以下两句没试，保存作用：
/usr/local/mysql/bin/mysqlbinlog --no-defaults --start-date='2016-10-28 05:00:00' --stop-date='2016-12-25 05:30:00' mysql-bin.000009 > restore.sql # 按时间点恢复
mysqlbinlog --stop-position="102" --start-position="367" mysql-bin.000001 | mysql -uroot -pxxx database_name # 按位置号恢复
```


#### blocked because of many connection errors
- blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'。
    - 原因：同一个ip在短时间内产生太多(超过mysql数据库max_connect_errors的最大值)中断的数据库连接而导致的阻塞
    - 解决办法1. cmd命令行：`mysqladmin flush-host -h 127.0.0.1 -u root -p123456`
    - 解决办法2. 提高允许的max_connect_errors数量
        - 查看max_connect_errors： `show variables like '%max_connect_errors%';`
        - 修改 `set global max_connect_errors = 1000;` 这样设置只对当前环境生效，重启就会失效，可以在my.cnf中设置
        - 或者修改配置文件/etc/my.cnf `max_connect_errors = 1000`
```mysql
use performance_schema;
select * from host_cache; # count_authentication_errors 记录账号密码错误的次数
# sum_connect_errors只计算协议握手错误，并且仅用于通过验证的主机(host_validated=yes)
```
错误原因：MySQL客户端与数据库建立连接需要发起三次握手协议，正常情况下，这个时间非常短，但是一旦网络异常，网络超时等因素出现，就会导致这个握手协议无法完成，MySQL有个参数connect_timeout，它是MySQL服务端进程mysqld等待连接建立完成的时间，单位为秒。如果超过connect_timeout时间范围内，仍然无法完成协议握手话，MySQL客户端会收到异常，异常消息类似于: Lost connection to MySQL server at 'XXX', system error: errno，该变量默认是10秒： 
`show variables like 'connect_timeout';`


### other

#### 事务日志
[refer](https://www.cnblogs.com/f-ck-need-u/archive/2018/05/08/9010872.html)

innodb事务日志包括redo log和undo log。redo log是重做日志，提供前滚操作，undo log是回滚日志，提供回滚操作。

undo log不是redo log的逆向过程，其实它们都算是用来恢复的日志：
- redo log通常是物理日志，记录的是数据页的物理修改，而不是某一行或某几行修改成怎样怎样，它用来恢复提交后的物理数据页(恢复数据页，且只能恢复到最后一次提交的位置)。
- undo用来回滚行记录到某个版本。undo log一般是逻辑日志，根据每行记录进行记录。

##### redo log

- redo log与二进制日志的区别
    - 二进制日志是在存储引擎的上层产生的，不管是什么存储引擎，对数据库进行了修改都会产生二进制日志。而redo log是innodb层产生的，只记录该存储引擎中表的修改。并且二进制日志先于redo log被记录。具体的见后文group commit小结。
    - 二进制日志记录操作的方法是逻辑性的语句。即便它是基于行格式的记录方式，其本质也还是逻辑的SQL设置，如该行记录的每列的值是多少。而redo log是在物理格式上的日志，它记录的是数据库中每个页的修改。
    - 二进制日志只在每次事务提交的时候一次性写入缓存中的日志"文件"(对于非事务表的操作，则是每次执行语句成功后就直接写入)。而redo log在数据准备修改前写入缓存中的redo log中，然后才对缓存中的数据执行修改操作；而且保证在发出事务提交指令时，先向缓存中的redo log写入日志，写入完成后才执行提交动作。
    - 因为二进制日志只在提交的时候一次性写入，所以二进制日志中的记录方式和提交顺序有关，且一次提交对应一次记录。而redo log中是记录的物理页的修改，redo log文件中同一个事务可能多次记录，最后一个提交的事务记录会覆盖所有未提交的事务记录
    - 事务日志记录的是物理页的情况，它具有幂等性，因此记录日志的方式极其简练。幂等性的意思是多次操作前后状态是一样的，例如新插入一行后又删除该行，前后状态没有变化。而二进制日志记录的是所有影响数据的操作，记录的内容较多。例如插入一行记录一次，删除该行又记录一次。
- MySQL支持用户自定义在commit时如何将log buffer中的日志刷log file中。这种控制通过变量 innodb_flush_log_at_trx_commit 的值来决定。该变量有3种值：0、1、2，默认为1。但注意，这个变量只是控制commit动作是否刷新log buffer到磁盘。
    - 当设置为1的时候，事务每次提交都会将log buffer中的日志写入os buffer并调用fsync()刷到log file on disk中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。
    - 当设置为0的时候，事务提交时不会将log buffer中日志写入到os buffer，而是每秒写入os buffer并调用fsync()写入到log file on disk中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。
    - 当设置为2的时候，每次提交都仅写入到os buffer，然后是每秒调用fsync()将os buffer中的日志写入到log file on disk
- log group
    - log group表示的是redo log group，一个组内由多个大小完全相同的redo log file组成。这个组是一个逻辑的概念。数量由`innodb_log_files_group`配置
- 日志刷盘的规则
    - 发出commit动作时。已经说明过，commit发出后是否刷日志由变量 innodb_flush_log_at_trx_commit 控制。
    - 每秒刷一次。这个刷日志的频率由变量 innodb_flush_log_at_timeout 值决定，默认是1秒。要注意，这个刷日志频率和commit动作无关。
    - 当log buffer中已经使用的内存超过一半时。
    - 当有checkpoint时，checkpoint在一定程度上代表了刷到磁盘时日志所处的LSN位置
##### undo log
- 基本概念
    - undo log有两个作用：提供回滚和多个行版本控制(MVCC)。
    - undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的update记录。
    - 当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。有时候应用到行版本控制的时候，也是通过undo log来实现的：当读取的某一行被其他事务锁定时，它可以从undo log中分析出该行记录以前的数据是什么，从而提供该行版本信息，让用户实现非锁定一致性读取。
    - undo log是采用段(segment)的方式来记录的，每个undo操作在记录的时候占用一个undo log segment。
    - 另外，undo log也会产生redo log，因为undo log也要实现持久性保护。
- 存储方式
- delete和update操作的内部机制
    - delete操作实际上不会直接删除，而是将delete对象打上delete flag，标记为删除，最终的删除操作是purge线程完成的。
    - update分为两种情况：update的列是否是主键列。
        - 如果不是主键列，在undo log中直接反向记录是如何update的。即update是直接进行的。
        - 如果是主键列，update分两部执行：先删除该行，再插入一行目标行
##### binlog和事务日志的先后顺序及group commit
- 但因为要保证二进制日志和事务日志的一致性，在提交后的prepare阶段会启用一个prepare_commit_mutex锁来保证它们的顺序性和一致性。但这样会导致开启二进制日志后group commmit失效，特别是在主从复制结构中，几乎都会开启二进制日志。
  
  在MySQL5.6中进行了改进。提交事务时，在存储引擎层的上一层结构中会将事务按序放入一个队列，队列中的第一个事务称为leader，其他事务称为follower，leader控制着follower的行为。虽然顺序还是一样先刷二进制，再刷事务日志，但是机制完全改变了：
  删除了原来的prepare_commit_mutex行为，也能保证即使开启了二进制日志，group commit也是有效的。
  
  MySQL5.6中分为3个步骤：flush阶段、sync阶段、commit阶段。
  
  
  
  flush阶段：向内存中写入每个事务的二进制日志。
  sync阶段：将内存中的二进制日志刷盘。若队列中有多个事务，那么仅一次fsync操作就完成了二进制日志的刷盘操作。这在MySQL5.6中称为BLGC(binary log group commit)。
  commit阶段：leader根据顺序调用存储引擎层事务的提交，由于innodb本就支持group commit，所以解决了因为锁 prepare_commit_mutex 而导致的group commit失效问题。
  在flush阶段写入二进制日志到内存中，但是不是写完就进入sync阶段的，而是要等待一定的时间，多积累几个事务的binlog一起进入sync阶段，等待时间由变量 binlog_max_flush_queue_time 决定，默认值为0表示不等待直接进入sync，设置该变量为一个大于0的值的好处是group中的事务多了，性能会好一些，但是这样会导致事务的响应时间变慢，所以建议不要修改该变量的值，除非事务量非常多并且不断的在写入和更新。
  
  进入到sync阶段，会将binlog从内存中刷入到磁盘，刷入的数量和单独的二进制日志刷盘一样，由变量 sync_binlog 控制。
  
  当有一组事务在进行commit阶段时，其他新事务可以进行flush阶段，它们本就不会相互阻塞，所以group commit会不断生效。当然，group commit的性能和队列中的事务数量有关，如果每次队列中只有1个事务，那么group commit和单独的commit没什么区别，当队列中事务越来越多时，即提交事务越多越快时，group commit的效果越明显。