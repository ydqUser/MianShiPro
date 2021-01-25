[TOC]

[refer(整体)](https://juejin.cn/post/6883270227078070286#heading-65)


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


### 配置
- 查看数据存放位置 `show variables like 'datadir';`
#### information_schema.tables
字段说明
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
[refer:分区]()


#### 存储架构
##### 区

##### 段
- 数据段
- 索引段
- 回滚段

##### 页

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

整理表碎片化空间方法:
- alter table tabname engine=InnoDB; # 
- ANALYZE table tabname;
- optimize table tabname;

#### 数据表分区
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

### 查询

### 索引

### 事务隔离级别

#### 事务

##### 一致性保证
[refer](https://blog.csdn.net/weixin_39781452/article/details/110864632)

#### 隔离级别

- 读未提交：导致脏读、不可重复读、幻读
- 读已提交：导致不可重复读、幻读
- 可重复读：**保证同一事务中多次读取相同数据的结果是一样的**。导致幻读

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

### 主从同步

主从复制：主从复制是指将主数据库的DDL和DML操作通过二进制日志传到从数据库上，然后在从数据库上对这些日志进行重新执行，从而使从数据库和主数据库的数据保持一致

原理：
- MySql主库在事务提交时会把数据变更作为事件记录在二进制日志Binlog中；
- 主库推送二进制日志文件Binlog中的事件到从库的中继日志Relay Log中，之后从库根据中继日志重做数据变更操作，通过逻辑复制来达到主库和从库的数据一致性；
- MySql通过三个线程来完成主从库间的数据复制，其中Binlog Dump线程跑在主库上，I/O线程和SQL线程跑着从库上；
- 当在从库上启动复制时，首先创建I/O线程连接主库，主库随后创建Binlog Dump线程读取数据库事件并发送给I/O线程，I/O线程获取到事件数据后更新到从库的中继日志Relay Log中去，之后从库上的SQL线程读取中继日志Relay Log中更新的数据库事件并应用，如下图所示

### 分库分表

### 8.0和5.7的区别

#### 隐藏索引
索引可以隐藏。隐藏索引对性能调试非常重要，索引可以被隐藏和显示。当一个索引隐藏时，不会被查询优化器所使用。
隐藏一个索引，然后观察数据库性能是否下降，如果下降，说明该索引有效，否则无效，可以删除

`ALTER TABLE t ALTER INDEX i INVISIBLE`
`ALTER TABLE t ALTER INDEX i VISIBLE`

#### 设置持久化

#### UTF-8编码

从mysql 8 开始，数据库的编码改为utf8mb4

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



