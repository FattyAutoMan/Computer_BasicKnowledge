# 索引

索引是帮助mysql高效获取数据的排好序的数据结构

索引数据结构

二叉树，红黑树，hash表，B-Tree（索引和数据放在一起）

Select * fron t where t.col2=89

 

B+数具有冗余数据，左边放的是小于，右边放的是大于等于

数据存储在叶子节点

 

InnoDB是聚集型索引

建议一定要建主键，使用整型的自增主键

非主键索引结构的叶子节点存储的是主键值，节省存储空间，保持一致性。

找到主键后再进行**主键索引**，**二次索引**。

 

Hash索引

对索引 key进行一次hash计算就可以定位出数据存储的位置；很多时候hash索引要比B+树索引更高效；

但不支持范围查询

存在hash冲突，mysql底层不会让链表特别长，通常会使用**再哈希**，mysql对哈希冲突解决较好。

 

# 事务

Innodb事务为例子

acid 原子性、一致性、隔离性、持久性

由undolog回滚用到、binlog、redolog保证一致性、持久性

undolog+redolog：原子性

redolog+binlog持久性

MVCC（版本链和read_view）实现隔离，redo_log（记录更新的sql是针对表的）实现原子，redo-log和binlog(mysql sever的)实现持久

隔离级别：读未提交（脏读）、读已提交（不可重复读）、可重复读、序列化

```
transaction Begin；//对删改的行增加排他锁；读的增加读锁
读和写操作完成，释放锁；//读未提交
rollback； //回滚
commit   //读已提交是在commit之后释放锁
```

mysql为了提高系统的**并发量**，在事务未提交前，虽然事务内操作的数据是锁定状态，但是另一个事务仍然可以读取。实现机制MVCC并发版本控制

# MVCC机制（版本链+快照）

主要针对读已提交（READ-COMMIT）和重复读两者隔离级别

主要是为了提高并发的读写性能，**不用加锁**就能让多个事务并发读写

实现原理：

在一个表中会增加tri_id事务id和指向上一条修改记录的指针

当执行查找sql                            

**版本链**

从上往下的执行顺序。

事务在增删更新的时候才会生成

查询select会产生一个**read_view**快照（活跃的未提交的事务id数组）

底层维护的**min_id**是未提交最小的，**max_id**是**已经创建**事务中最大（可能提交或者没提交）

从版本链中当前的事务id找起

 <img src="C:\Users\llh\AppData\Roaming\Typora\typora-user-images\image-20210909160839672.png" alt="image-20210909160839672" style="zoom:50%;" />

每次更新都会把之前的记录放到undo日志种

注意undo日志就一份，通过read_view中min_id和max_id进行规则匹配，同一个select

可能只有一个readview存在更新不及时

Read_view是每个select会生成一份，这个快照是针对全库的，所以数据可能不对

 

# 索引底层执行原理

对abcd字段的表建立bcd联合索引，只是截取bcd部分数据进行排序，然后建立B+树。

为了检索所整条信息，保存对于的主键；通过二次搜索（回表）找到整个数据记录。 

记录主键，回表

# 最左匹配原理

bcd联合索引，走索引的右b,bc, bcd

了解上面bcd索引建立底层就很好理解

 

索引优化策略

Explain select * from t1 where b>1; 可以走索引但是走全表扫描会快

 

# 索引失效

Sql会把数字字符转换成相应数字；把非数字字符转换成0.

**Create index** idx_e **on** t1(e) // e是字符串

Create index idx_t1_bcd on t1(b,c,d) // abcd都是数字

Create index idx_t1_bcd on t1(b desc, c asc, d) //5.8之后，可降序

E**xplain select** * f**rom** t1 wh**ere** e=1;//不会走索引，因为需要把e中的所有字母转成0

Explain select * from t1 where a=’2a’ //‘2a’转换成0会走所有 

Explain select * from t1 where e=’1’ //‘1‘转换成1会走索引

Explain select * from t1 where a=1 //会走索引

Explain select * from t1 order by b desc c, d //如果是按照升序建立的索引，那么不会走索引

 

# Explain

explain这个命令来查看一个这些SQL语句的执行计划，查看该SQL语句有没有使用上了索引，有没有做全表扫描

https://www.cnblogs.com/leeego-123/p/11846613.html

 

# join

https://www.cnblogs.com/leeego-123/p/11846613.html

https://www.cnblogs.com/yixianyixian/p/9336840.html

nature join 

inner join

outer join 

cross join

表的关联查询，比如电商的customer表和order表中联合查询改用户购物车中的商品

# buff pool 内存结构

#  

 

 

 

三个链表维护：空闲的，已经加载的；脏块

 

## 持久性实现以及为什么引如redo_log

\1.   修改buffer pool中的页数据---脏页

\2.   Update语句 --à 生成一个reddo log对象

\3.   Redo log持久化

\4.   修改成功

Mysql挂了 重启之后会从磁盘和redolog中找到原有数据

 

\1.   修改buffer pool中的页数据 ---脏页

\2.   修改磁盘页是数据      随机IO读写代价大 

\3.   修改成功

 

Redo log就是不用每次修改都进行一次随机IO，可以记录后一次性进行磁盘持久化

当事务提交的时候进行磁盘持久化，回滚就不需要了。

 

Redolog一般由系统在空闲时输入磁盘，

一个事务开始，redolog写盘，事务进入准备阶段

完成准备阶段，binlog写盘，并刷入磁盘，实现持久化

成功则在redo log中标志commit

# Write和flush

Write是写内存

Flush是写磁盘

# Redo_log（是innodb中的概念）文件的持久化

Innodb_flush_log_at_trx_conmmit 设置项

0：事务提交时，不立即对redo log进行持久化，这个任务交给后台线程做，

1：表示事务提交时，立刻把redo log进行持久化。先执行write然后执行flush

3：表示事务提交时，立即将redo log写到操作系统的缓冲区，并不会直接将redo log纪念性持久化，这种情况下，如果数据库挂了，但是操作系统没挂，那么试问持久性还是可以保证的（只是写write，接着由操作系统管理进行flush）

 

 

Bin log和undolog是mysql概念

Redo log记录的是某一页中被修改数据的物理位置

Binlog记录了是update sql语句

Mysql数据库的复制要用binlog

因为redo log只有48M大小有限

Undolog是反向更新日志

Rollback会用到

 

# 读写分离

\1.   高并发，在数据库层面进行读写分离优化

\2.   换成mongodb

\3.   Sql字段你，索引

\4.   分库分表

 

## 通过binlog进行主从同步

主要是slave中I/O线程进行binlog拷贝，sql线程进行读

 

## 强制路由

Slave延迟导致没有及时同步数据，这个时候强制从master中读数据。

在sql前加上master即可

 

# 分库分表

垂直拆分：拆分的是字段

表结构不一样；但是有一项字段相同

优点：业务清晰；数据维护简单、按业务不同放在不同的业务

缺点：单表数据大压力大；受到业务比如双十一购物。订单业务会影响到数据库瓶颈

   部分业务无法关联join，只能通过java程序端口区调用，提高了开发难度

 

水平分库/表

拆分的是内容。

优点：负载均衡性能高，不存在瓶颈；

​    切分的表的数据结构一样；

缺点：数据扩容难（如果数据库增加需要进行数据迁移）；

   拆分规则；

   分片事务的一致性的问题部分业务无法关联join，只能通过java程序接口去调用

​    如果只有一个数据库只需要遵循它的acid，但是如果多个数据库就会涉及到acid分布业务（这个只要是分库就会村子啊）

 

# 跨库查询问题

问题: 主建避重；公共表处理；事务一致性；跨节点关联查询；跨节点分页、排序函数

实战解决方案:

Proxy代理层：mycat, altas, mysql-proxy，shardingproxy

Jdbc应用层: **shardingsphare****(****生态圈)**, TDDL（淘宝的不完全开源）,shardingjdbc

 

 

代理层是通过网络通信实现的，性能差，但是跨语言；不支持跨数据库

Jdbc是只能用java语言，但是性能高；跨数据库

 

# Shardsphare(生态圈)分库分表实战

https://www.bilibili.com/video/BV1xh411Z79d?p=29&spm_id_from=pageDriver

 

支支持mysql和pgsql

分库分表的注册中心怎么用？？？

主键避重：雪花算法

 

# 事务隔离级别

Undolog实现

# Mysql聚簇索引和非聚簇索引的区别

数据存储是否和索引放在一块

优点：

①  聚簇索引索引和数据放在一起。聚集索引可以直接获得数据，非聚簇需要二次查询数据地址进行io。查询效率更高

②  聚簇索引相邻的索引对应的数据也是放在一起的。而非聚簇所以数据 不一定放在一起。所以范围搜索数据效率高

③  聚簇索引适合用在排序的场合

劣势：

①  插入新数据的时候，在中间插入，后面的数据需要发生移动，维护昂贵

②  索引文件大，占用内存大

数据量大，全文扫描多的myisam更好

# 索引的设计原则

查询更快，占用空间更小

①  适合出现在where子句中的列

②  基数较小的字段。表的数据本来就不多的时候。不如全表扫描

③  使用短索引。对于长字符串应该指定一个前缀长度。节省空间

④  不要过度索引

⑤  定义由外键的数据列一定要建立索引

⑥  更新频繁的，不建议建立索引

⑦  不能有效区分数据。比如性别男女

⑧  尽量拓展索引，不要新建索引。如已经由a索引了，现在加（a,b）联合索引，修改原来索引为联合索引。不要单独建立b索引

⑨  实际查询很少用到的列

⑩  定义为text，image和bit数据

 

# Mysql锁的类型有哪些

属性分类：共享锁，排他锁

粒度分类：行级，表级，页级，行级

共享锁（share Lock）简称S锁：当一个事务为数据加上读锁之后，其他事务只能对该数据加读锁，而不能对数据加写锁。共享锁是为了文件并发地读数据。只有当所有读锁都释放了之后才能对数据加写锁。

排他锁(exclusive lock) 简称X锁，排他锁又称写锁，当一个事务为数据加上写锁的时候，其他请求不能为数据加任何锁。

表锁：对整个表锁住。加锁简单，容易冲突

行锁：

记录锁：只会锁一条（唯一索引），精准命中。避免重复读和脏读

页锁：介于表锁和行锁之间

间隙锁：锁住一个区间。行锁可能锁住1，4，间隙会把2，3（1，4）锁住

临界锁：间隙+行锁【1，4】

意向共享锁：对行进行锁，就会对表加上意向锁。提高加锁效率，不用加锁的时候都扫描一遍。

意向排他锁：

 

# Mysql执行计划怎么看

用explian就可以知道执行计划https://www.cnblogs.com/leeego-123/p/11846613.html

要知道内容

Id：select序号

selectType: SIMPLE,PRIMARY,SUBQUERY,UNION,DEPENDENT, DERIVED

table:

partitions:

type（索引类型和效率有关）：const（通过索引一次命中，匹配一行数据）, system（表中只有一行记录，相当于系统表）,eq_ref(唯一索引扫描)， ref（非唯一索引），range(范围查询)，index（遍历索引树，遍历所有），all(表示全表扫描)

possible_keys: 可能会走的索引

key: 实际走的索引名字

key_len: 联合索引中，命中的索引个数

ref: 命中索引的字段名，数据库.表名.字段名

rows: 读取数据量（多少行），行数越少执行计划越优

filter: 过滤，百分比，返回行数/读取行数

extra：using_filesort 没走索引

​    using index 使用索引

​    using_temporary使用临时表（说明效率不高）

# 事务的概念和acid

原子性：要门更新成功，要么回滚

一致性：数据库总是会从一个一致性状态转换到另外一个一致性的状态。

隔离性：

持久性：磁盘持久化

实际场景：A向B转账100.A账户少100, B账户多100.

​     A原来500/B 100àA 400/B 200

​     状态只能是这两者一个，保证一致性

要么发生转移要么保持原样---- 原子性

C也要给B转100，A也给B转可能事务冲突（事务同步）--- 隔离性

实际操作会分配事务id(tr_id)从而确定了事务执行的先后顺序

事务一旦提交，所做的修改会永久保存在数据库，不能因为故障而丢失数据---持久性

 

A实际只有90，在转移前数据符合约束（不能出现负数），如果事务成功执行，就会破坏约束，因此事务不能成功。这里就保证两个状态状态在满足约束情况下保持状态一致。**强调的是一种目的。**

 

## 事务的隔离级别

Read_uncmmit：读未提交。因为未提交的数据可能最终会回滚，就出现了脏读

Read_commit: 读已提交。每次select都会生成一个read_view。解决了脏读问题。存在不可重复读。由于某个数据被改变了，所有上一次读和这一次读数据不一样。（java的话可以加读锁解决）

Repeat_read: 重复读。通过只生成一个read_view快照解决了不可重复读。但是会存在幻读。

快照的话指挥保存第一次select的快照，但是如果插入，快照数据就会增多。这就是幻读

Serialization串读：一般是不会使用的，他会给每一行读取的数据加锁，会导致大量超时和锁竞争问题。

 

脏读：就是没提交数据读取

不可重复度：数据被修改，前后读取的数据不一样

幻读：虽然是一个read_view,，但是插入新数据，会造成笔数不对。

# 关心过业务系统里面sql耗时吗？统计过慢查询吗？点对慢查询都怎么优化过？

在业务系统中如果只是使用主键进行查询，那么久没有什么优化余地。

慢查询的统计一般由运维在做会定期把慢查询反馈给程序员。

慢查询产生原因：①查询条件没有命中 ②load了不必要的数据列实际只要5行却select全部③数据量太大了（分库分表）

 

# ACID靠什么保证的？

原子性，一致性，隔离性，持久性

Innodb和myisam

在innodb有redolog和undolog

在mysql数据库层面偶binlog

**原子性**是由 undolog保证回滚

一致性由其它三个保证+程序代码保证业务一致性

**隔离性**是由**MVCC**机制（在不加锁的实现高并发），这样读写都可以不用加锁

**持久性**由内存+redolog来保证。在mysql sever级别由binlog记录sql语句（主从同步也是通过读取binlog）

## 如何保证binlog和redolog一致？

Innodb redo_log写盘，innodb事务进入prepare阶段

如果prepare成功，binlog写盘，在继续将事务日志持久化到binlog。如果持久化成功，那么innodb事务则进入commit状态（在redo_log里面写一个commit记录）

 

如果一个事务在redo_log中由commit标志及说明binlog完成持久化成功。‘

Commit成功也就是binlog持久化成功

 

Redo_log刷盘会在系统空闲时候进行。

 

## 什么是MVCC？

多版本并发控制 multi version control

版本链条，redolog, undolog, read_view(存放活动事务的id数组, min_id和max_id)

读取数据时通过一种类似快照的方式将数据保存下来，这样读锁和写锁就不冲突了，不同的事务seesion只会看到自己特定版本的数据，版本链

MVCC只是在read committed和 repeatable read两个隔离级别下工作。其他两个隔离级别和MVCC不兼容，因为read uncomit总是读取最新的数据行，而不是符合当前事务版本的数据行。而seraializable则会对所有读取的行加锁。

 

聚集索引记录中会由两个必要的列； trx_id事务id和roll_pointer指向上一个版本指针

 

# Mysql主从同步原理

Master slave

三个线程：主库binlog _dump_thread ,从库（i/o thread sql_thread

 

 

# 简述MYISAM和innodb的区别

 

# 简述mysql中索引类型及对数据库的性能的影响

普通索引（数据列内容可以重复）

唯一索引

主键

联合索引

全文索引

 

全文索引一般会使用ES技术提高性能

 

# 为什么索引不能滥用？

索引在增删更新的时候也是要维护的；

索引占有物理空间。