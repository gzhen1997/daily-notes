#### 1.mysql体系结构

![ ](C:\Users\15444\Desktop\bj\images\image-20220308225040407.png)

#### 2.InnoDB存储引擎

##### 特点

​     1. DML操作遵循ACID模型，支持事务

​     2. 行级锁，提高并发访问性能

​     3. 支持外键约束，保证数据的完整性和正确性

##### 文件

​        xx.idb： xxx代表的是表名，innoDB引擎的每张表都会对应这样一个表的空间文件，存储该表的表结构（frm、sdi）、数据和索引。

​        可以通过查看mysql系统变量来确定是否每一张表都会产生一个表空间文件 `show variables like 'innodb_file_per_table'`

​        查看xxx.idb命令 `ibd2sdi xxx.idb`

#####    存储结构（逻辑存储结构）

​        TableSpace：表空间

​        Segment： 段

​        Extent：区

​        Page： 页

​        Row： 行

​        **一个区1M 一个Page16K  一个区可以包含64个页**

![image-20220308232329774](C:\Users\15444\Desktop\bj\images\image-20220308232329774.png)                        

#### 3.MyISAM存储引擎

​       MyISAM是mysql早期的默认存储引擎

#####  特点

​      1.不支持事务，不支持外键

​      2.支持表锁，不支持行锁

​      3.访问速度快

#####  文件

​      xxx.sdi：存储表结构文件

​      xxx.MYD：存储数据

​      xxx.MYI：存储索引

#### 4.Memory存储引擎

​      Memory引擎的表数据存储在内存中，由于受硬件问题、或断电问题的影响，只能将这些表作为临时表或者缓存使用

#####  特点

​      1.内存存放

​      2.hash索引（默认）

#####  文件

​      xxx.sdi:存储表结构信息

#### 5.存储引擎的区别

![image-20220308233051583](C:\Users\15444\Desktop\bj\images\image-20220308233051583.png)

#### 6.存储引擎的选择

![image-20220308233437422](C:\Users\15444\Desktop\bj\images\image-20220308233437422.png)

#### 7.SQL性能分析

##### profile详情（慢查询记录不了的记录）

   `show profiles`能够在做sql优化时帮助我们了解时间消耗到哪里去了。通过 `hava_profiling`参数，能够看到当前mysql是否支持profile操作

   `select @@have_profiling;`  开启profiles  `set profiling=1;`

#####  Explain/desc执行计划

   **Explain** 或者**des**命令获取mysql如何执行select语句的信息，包括在select语句执行过程中如何连接和什么样的连接顺序

   语法：直接在select语句前面加上**Explain**或者**desc** 。例如 `explain select * from user;` 

​	![image-20220310223909626](C:\Users\15444\Desktop\bj\images\image-20220310223909626.png)			 

​		![image-20220310230805579](C:\Users\15444\Desktop\bj\images\image-20220310230805579.png)				![image-20220310231017657](C:\Users\15444\Desktop\bj\images\image-20220310231017657.png)

 

#### 8.锁

##### 全局锁

**![image-20220310234956846](C:\Users\15444\Desktop\bj\images\image-20220310234956846.png)**

`flush tables with read lock; `添加全局锁

`mysqldump -uroot -p123456 itcast > itcast.sql;` 数据备份

`unlock tables;` 释放全局锁

![image-20220311000222997](C:\Users\15444\Desktop\bj\images\image-20220311000222997.png)

   **通过快照度来实现的**

##### 表级锁

![image-20220311000507275](C:\Users\15444\Desktop\bj\images\image-20220311000507275.png)



###### 表锁

![image-20220311000616109](C:\Users\15444\Desktop\bj\images\image-20220311000616109.png)	

   读锁：不会阻塞其他客户端的读操作，但是会阻塞所有客户端的写操作。

   写锁：当前客户端可以读也可以写，其他客户端的读写操作均被阻塞。

###### 元数据锁

![image-20220311001945098](C:\Users\15444\Desktop\bj\images\image-20220311001945098.png)

![image-20220311002557958](C:\Users\15444\Desktop\bj\images\image-20220311002557958.png)

###### 意向锁

![image-20220311003415876](C:\Users\15444\Desktop\bj\images\image-20220311003415876.png)	

![](C:\Users\15444\Desktop\bj\images\image-20220311003459915.png)

![image-20220311003537042](C:\Users\15444\Desktop\bj\images\image-20220311003537042.png) 

**![image-20220311003235897](C:\Users\15444\Desktop\bj\images\image-20220311003235897.png)**

##### 行级锁

![image-20220315202813192](C:\Users\15444\Desktop\bj\images\image-20220315202813192.png)

![image-20220315203004683](C:\Users\15444\Desktop\bj\images\image-20220315203004683.png)

![image-20220315203028604](C:\Users\15444\Desktop\bj\images\image-20220315203028604.png)

![image-20220315203214384](C:\Users\15444\Desktop\bj\images\image-20220315203214384.png)

![image-20220315203905252](C:\Users\15444\Desktop\bj\images\image-20220315203905252.png)

![image-20220315210127857](C:\Users\15444\Desktop\bj\images\image-20220315210127857.png)

![image-20220315210736361](C:\Users\15444\Desktop\bj\images\image-20220315210736361.png)

#### 9.InnoDB存储引擎详解

##### 逻辑存储结构 

![image-20220315214234650](C:\Users\15444\Desktop\bj\images\image-20220315214234650.png)

表空间：idb文件，一个mysql实例可以对应多个表空间，用于存储记录、索引等数据。

段：分为**数据段（Leaf node_segment）**、**索引段（Non-leaf node segment）**、**回滚段（Rollback segment）**，InnoDB是索引组织表，数据段就是B+树的叶子节点，索引段即为B+树的非叶子节点。用来管理多个Extent（区）。

区：表空间的单元结构，每个区的大小为1M。默认情况下，InnoDB存储引擎页大小为16K，即一个区中一个有64个连续的页。

页：是**InnoDB存储引擎磁盘管理的最小单元**，每个页默认大小为16KB，为了保证页的连续性，InnoDB存储引擎每次从磁盘申请4-5个区。

行：InnoDB存储引擎数据是按行存放 的。

​        Trx：每次对某条数据进行改动时，都会把对应的事务id赋值给trx_id隐藏列。

​        Roll_pointer：每次对某条记录进行改动时，都汇报旧的版本写入到undo日志中，然后这个隐藏列就相当于一个指针，可以通过它来找到该记录修改的信息

##### 架构

![image-20220315215613506](C:\Users\15444\Desktop\bj\images\image-20220315215613506.png)

###### 内存结构

![image-20220315220102239](C:\Users\15444\Desktop\bj\images\image-20220315220102239.png)

 ![image-20220315220439689](C:\Users\15444\Desktop\bj\images\image-20220315220439689.png)

![image-20220315220707251](C:\Users\15444\Desktop\bj\images\image-20220315220707251.png)

![image-20220315220916489](C:\Users\15444\Desktop\bj\images\image-20220315220916489.png)

######  磁盘结构

![image-20220315222513509](C:\Users\15444\Desktop\bj\images\image-20220315222513509.png)

​    ![image-20220315222622103](C:\Users\15444\Desktop\bj\images\image-20220315222622103.png)

![image-20220315222121584](C:\Users\15444\Desktop\bj\images\image-20220315222121584.png)

###### 后台线程

![image-20220315223504776](C:\Users\15444\Desktop\bj\images\image-20220315223504776.png)

查看线程的执行情况 `show engine innodb status`

#### 10.事务

![image-20220315224313438](C:\Users\15444\Desktop\bj\images\image-20220315224313438.png)

![image-20220315224520901](C:\Users\15444\Desktop\bj\images\image-20220315224520901.png)

##### 事务原理

###### redo log

![image-20220321213932330](C:\Users\15444\Desktop\bj\images\image-20220321213932330.png)

![image-20220321214743668](C:\Users\15444\Desktop\bj\images\image-20220321214743668.png)

 总结：

   redo log 可以防止提交的事务重操作多个数据页导致大量的随机io从而影响mysql服务的效率问题，由于redo log在每次提交事务的时候都往redo log file中追加数据，所以他是顺序io，他的效率非常高。

###### undo log

![image-20220321215822533](C:\Users\15444\Desktop\bj\images\image-20220321215822533.png)

#### 11.MVC

##### 概念

![image-20220321221643013](C:\Users\15444\Desktop\bj\images\image-20220321221643013.png)

##### 原理

######  隐藏字段

![image-20220321221850750](C:\Users\15444\Desktop\bj\images\image-20220321221850750.png)

注意：如果当前表有主键，DB_ROW_ID就不会出现。

###### 版本链

**![image-20220321224843980](C:\Users\15444\Desktop\bj\images\image-20220321224843980.png)**



###### readview（视图读）

​       决定选择undo log版本链中的具体数据

![image-20220321225208886](C:\Users\15444\Desktop\bj\images\image-20220321225208886.png)

![image-20220321225745326](C:\Users\15444\Desktop\bj\images\image-20220321225745326.png)

![image-20220321225817777](C:\Users\15444\Desktop\bj\images\image-20220321225817777.png)

RC(读已提交Read Commit)

![image-20220321231012068](C:\Users\15444\Desktop\bj\images\image-20220321231012068.png)

RR(可重复读Repeatable Read)

![image-20220321231225769](C:\Users\15444\Desktop\bj\images\image-20220321231225769.png)

![image-20220321231359786](C:\Users\15444\Desktop\bj\images\image-20220321231359786.png)