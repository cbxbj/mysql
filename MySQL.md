# MySQL

> 注：文档中所有内容及SQL基于8.x版本，5.x版本运行可能会存在不正确问题

## 锁

### 概述

锁是计算机协调多个进程或线程`并发访问某一资源`的机制。在数据库中，除传统的计算资源（CPU、 RAM、I/O）的争用以外，数据也是一种供许多用户共享的资源。如何保证数据并发访问的一致性、有 效性是所有数据库必须解决的一个问题，锁冲突也是影响数据库并发访问性能的一个重要因素。从这个 角度来说，锁对数据库而言显得尤其重要，也更加复杂。

MySQL中的锁，按照锁的粒度分，分为以下三类： 

- `全局锁`：锁定数据库中的所有表。 
- `表级锁`：每次操作锁住整张表。 
- `行级锁`：每次操作锁住对应的行数据。

### 全局锁

#### 概述

全局锁就是对整个数据库实例加锁，加锁后整个实例就处于只读状态，后续的DML的写语句，DDL语 句，已经更新操作的事务提交语句都将被阻塞。 其典型的使用场景是做全库的逻辑备份，对所有的表进行锁定，从而获取一致性视图，保证数据的完整 性。

#### 数据备份语法

##### 1.加全局锁

```mysql
-- 1.加全局锁
flush tables with read lock ;
```

```mysql
-- 2.数据备份
-- 注：该命令在cmd下运行
-- mysqldump -u用户名 –p密码 数据库名 > 路径
mysqldump -uroot –p123456 test > D:/test.sql
```

```mysql
-- 3.释放锁
unlock tables ;
```

##### 2.通过快照

```mysql
-- 注：该命令在cmd下运行
mysqldump --single-transaction -uroot –p123456 test > D:/test.sql
```

### 表级锁

#### 概述

表级锁，每次操作锁住整张表。锁定粒度大，发生锁冲突的概率最高，并发度最低。应用在MyISAM、 InnoDB、BDB等存储引擎中。

对于表级锁，主要分为以下三类：

- 表锁
- 元数据锁（meta data lock，MDL）
- 意向锁

#### 表锁

##### 概述

对于表锁，分为两类： 

- 表共享读锁（read lock）
- 表独占写锁（write lock）

读锁不会阻塞所有客户端的读，但是**会阻塞所有客户端的写**。

写锁**只会阻塞其他客户端的读和写**，不会阻塞自身的读和写。

##### 语法

- 加锁：`lock tables 表名... read/write`。
- 释放锁：`unlock tables` / 客户端断开连接 。

#### 元数据锁

##### 概述

meta data lock , 元数据锁，简写`MDL`。

MDL加锁过程是**系统自动控制**，无需显式使用，在访问一张表的时候会自动加上。MDL锁主要作用是维 护表元数据的数据一致性，在表上有活动事务的时候，不可以对元数据进行写入操作。**为了避免DML与 DDL冲突，保证读写的正确性**。

这里的元数据，大家可以简单理解为就是一张表的表结构。 也就是说，某一张表涉及到未提交的事务 时，是不能够修改这张表的表结构的。

在MySQL5.5中引入了MDL，当对一张表进行增删改查的时候，加MDL读锁(共享)；当对表结构进行变 更操作的时候，加MDL写锁(排他)。

| 对应SQL                                         | 锁类型                                  | 说明                                               |
| ----------------------------------------------- | --------------------------------------- | -------------------------------------------------- |
| lock tables xxx read / write                    | SHARED_READ_ONLY / SHARED_NO_READ_WRITE |                                                    |
| select 、select ... lock in share mode          | SHARED_READ                             | 与SHARED_READ、 SHARED_WRITE兼容，与 EXCLUSIVE互斥 |
| insert 、update、 delete、select ... for update | SHARED_WRITE                            | 与SHARED_READ、 SHARED_WRITE兼容，与 EXCLUSIVE互斥 |
| alter table ...                                 | EXCLUSIVE                               | 与其他的MDL都互斥                                  |

##### 查看元数据锁

```mysql
select object_type,object_schema,object_name,lock_type,lock_duration from
performance_schema.metadata_locks ;
```

#### 意向锁

##### 概述

为了DML在执行时，加的**行锁与表锁不冲突**，在InnoDB中引入了意向锁，使得表锁不用检查每行 数据是否加锁，使用意向锁来减少表锁的检查。

对于意向锁，分为两类： 

- 意向共享锁(IS): 由语句select ... lock in share mode添加 。 与 表锁共享锁 (read)兼容，与表锁排他锁(write)互斥。
- 意向排他锁(IX): 由insert、update、delete、select...for update添加 。与表锁共 享锁(read)及排他锁(write)都互斥，意向锁之间不会互斥。

##### 查看意向锁/行锁语句

```mysql
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
performance_schema.data_locks;
```

### 行级锁

#### 概述

行级锁，每次操作锁住对应的行数据。锁定粒度最小，发生锁冲突的概率最低，并发度最高。应用在 `InnoDB`存储引擎中。

InnoDB的数据是基于索引组织的，**行锁是通过对索引上的索引项加锁来实现的，而不是对记录加的锁**。对于行级锁，主要分为以下三类：

- 行锁（Record Lock）：锁定单个行记录的锁，防止其他事务对此行进行update和delete。在 **RC、RR隔离级别**下都支持。

  ![](https://github.com/cbxbj/mysql/blob/master/img/001.png)

- 间隙锁（Gap Lock）：锁定索引记录间隙（不含该记录），确保索引记录间隙不变，防止其他事 务在这个间隙进行insert，产生幻读。在**RR隔离级别**下都支持。

  ![](https://github.com/cbxbj/mysql/blob/master/img/002.png)

- 临键锁（Next-Key Lock）：行锁和间隙锁组合，同时锁住数据，并锁住数据前面的间隙Gap。 在**RR隔离级别**下支持。

  ![](https://github.com/cbxbj/mysql/blob/master/img/003.png)

#### 行锁

##### 概述

InnoDB实现了以下两种类型的行锁：

- 共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同数据集的排它锁。
- 排他锁（X）：允许获取排他锁的事务更新数据，阻止其他事务获得相同数据集的共享锁和排他锁。

|             | 共享锁（S） | 排他锁（X） |
| ----------- | ----------- | ----------- |
| 共享锁（S） | 兼容        | 冲突        |
| 排他锁（X） | 冲突        | 冲突        |

常见的SQL语句，在执行时，所加的行锁如下：

| SQL                           | 行锁类型       | 说明                                     |
| ----------------------------- | -------------- | ---------------------------------------- |
| INSERT ...                    | 排他锁         | 自动加锁                                 |
| UPDATE ...                    | 排他锁         | 自动加锁                                 |
| DELETE ...                    | 排他锁         | 自动加锁                                 |
| SELECT（正常）                | **不加任何锁** |                                          |
| SELECT ... LOCK IN SHARE MODE | 共享锁         | 需要手动在SELECT之后加LOCK IN SHARE MODE |
| SELECT ... FOR UPDATE         | 排他锁         | 需要手动在SELECT之后加FOR UPDATE         |

- 针对唯一索引进行检索时，对已存在的记录进行等值匹配时，将会自动优化为行锁。
- InnoDB的行锁是针对于索引加的锁，**不通过索引条件检索数据**，那么InnoDB将对表中的所有记录加锁，此时 就会**升级为表锁**。

##### 查看意向锁/行锁语句

```mysql
select object_schema,object_name,index_name,lock_type,lock_mode,lock_data from
performance_schema.data_locks;
```

#### 间隙锁/临键锁

##### 概述

默认情况下，InnoDB在 REPEATABLE READ事务隔离级别运行，InnoDB使用 `next-key lock(临键锁)`进行搜 索和索引扫描，以防止幻读。

表中有id=1、6，8，9的值

- 索引上的等值查询(**唯一索引**)，给不存在的记录加锁时, 优化为**间隙锁** 。
  - 锁的是当前数值，到上一个数值的间隙，开始事务后，更新id=8的值，则间隙锁，锁住(6,8)的间隙
- 索引上的等值查询(**非唯一普通索引**)，向右遍历时最后一个值不满足查询需求时，next-key lock 退化为间隙锁。
  - 锁的是当前数值，到上一个数值的间隙，以及到下一个数值的间隙，开始事务后，更新id=8的值，则间隙锁，锁住(6,8)的间隙、(8，9)的间隙
- 索引上的范围查询(唯一索引)--会访问到不满足条件的第一个值为止。
  - 锁的是当前数值到上个数值间隙，以及范围的值

> 注：间隙锁唯一目的是防止其他事务插入间隙。间隙锁可以共存，一个事务采用的间隙锁不会 阻止另一个事务在同一间隙上采用间隙锁。

## InnoDB

### 逻辑存储结构

#### 图示

![](https://github.com/cbxbj/mysql/blob/master/img/004.png)

![](https://github.com/cbxbj/mysql/blob/master/img/005.png)

#### 详解

##### 表空间

表空间是InnoDB存储引擎逻辑结构的最高层， 如果用户启用了参数 innodb_file_per_table(在 8.0版本中默认开启) ，则每张表都会有一个表空间（xxx.ibd），一个mysql实例可以对应多个表空间，用于**存储表结构、索引、数据**等数据。

##### 段

段，分为数据段（Leaf node segment）、索引段（Non-leaf node segment）、回滚段 （Rollback segment），InnoDB是索引组织表，数据段就是B+树的叶子节点， 索引段即为B+树的 非叶子节点。段用来管理多个Extent（区）。

##### 区

区，表空间的单元结构，每个区的大小为1M。 默认情况下， InnoDB存储引擎页大小为16K， 即一 个区中一共有64个连续的页。

##### 页

页，是InnoDB 存储引擎磁盘管理的最小单元，每个页的大小默认为 16KB。为了保证页的连续性， InnoDB 存储引擎每次从磁盘申请 4-5 个区。

##### 行

行，InnoDB 存储引擎数据是按行进行存放的。

在行中，默认有两个隐藏字段：

- Trx_id：每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列。
- Roll_pointer：每次对某条引记录进行改动时，都会把旧的版本写入到undo日志中，然后这个 隐藏列就相当于一个指针，可以通过它来找到该记录修改前的信息。

### 架构

#### 概述

MySQL5.5 版本开始，默认使用InnoDB存储引擎，它擅长事务处理，具有崩溃恢复特性，在日常开发 中使用非常广泛。下面是InnoDB架构图，左侧为内存结构，右侧为磁盘结构。

![](https://github.com/cbxbj/mysql/blob/master/img/006.png)

#### 内存结构

在左侧的内存结构中，主要分为这么四大块儿： 

- Buffer Pool
- Change Buffer
- Adaptive Hash Index
- Log Buffer

##### Buffer Pool

![](https://github.com/cbxbj/mysql/blob/master/img/007.png)

InnoDB存储引擎基于磁盘文件存储，访问物理硬盘和在内存中进行访问，速度相差很大，为了尽可能 弥补这两者之间的I/O效率的差值，就需要把经常使用的数据加载到缓冲池中，避免每次访问都进行磁 盘I/O。

在InnoDB的缓冲池中不仅缓存了索引页和数据页，还包含了undo页、插入缓存、自适应哈希索引以及 InnoDB的锁信息等等。

缓冲池 Buffer Pool，是主内存中的一个区域，里面可以缓存磁盘上经常操作的真实数据，在执行增 删改查操作时，先操作缓冲池中的数据（若缓冲池没有数据，则从磁盘加载并缓存），然后再以一定频 率刷新到磁盘，从而减少磁盘IO，加快处理速度。

缓冲池以Page页为单位，底层采用**链表**数据结构管理Page。根据状态，将Page分为三种类型：

- free page：空闲page，未被使用。  
- clean page：被使用page，数据没有被修改过。 
- dirty page：脏页，被使用page，数据被修改过，也中数据与磁盘的数据产生了不一致。

在专用服务器上，通常将多达**80％的物理内存分配给缓冲池**

###### 参数

```mysql
show variables like 'innodb_buffer_pool_size';
```

##### Change Buffer

注在5.x版本无此概念，5.x有插入缓冲区(Insert Buffer),8.0有Change Buffer

![](https://github.com/cbxbj/mysql/blob/master/img/008.png)

Change Buffer，更改缓冲区（针对于非唯一二级索引页），在执行DML语句时，如果这些数据Page 没有在Buffer Pool中，不会直接操作磁盘，而会将数据变更存在更改缓冲区 Change Buffer 中，在未来数据被读取时，再将数据合并恢复到Buffer Pool中，再将合并后的数据刷新到磁盘中。

![](https://github.com/cbxbj/mysql/blob/master/img/009.png)

与聚集索引不同，二级索引通常是非唯一的，并且以相对随机的顺序插入二级索引。同样，删除和更新 可能会影响索引树中不相邻的二级索引页，如果每一次都操作磁盘，会造成大量的磁盘IO。有了 ChangeBuffer之后，我们可以在缓冲池中进行合并处理，减少磁盘IO。

##### Adaptive Hash Index

![](https://github.com/cbxbj/mysql/blob/master/img/010.png)

自适应hash索引，用于优化对Buffer Pool数据的查询。MySQL的innoDB引擎中虽然没有直接支持 hash索引，但是给我们提供了一个功能就是这个自适应hash索引。因为前面我们讲到过，hash索引在 进行等值匹配时，一般性能是要高于B+树的，因为hash索引一般只需要一次IO即可，而B+树，可能需 要几次匹配，所以hash索引的效率要高，但是hash索引又不适合做范围查询、模糊匹配等。

InnoDB存储引擎会监控对表上各索引页的查询，如果观察到在特定的条件下hash索引可以提升速度， 则建立hash索引，称之为自适应hash索引。

**自适应哈希索引，无需人工干预，是系统根据情况自动完成。**

###### 参数

 `innodb_adaptive_hash_index`

```mysql
show variables like 'innodb_adaptive_hash_index'
```

##### Log Buffer

![](https://github.com/cbxbj/mysql/blob/master/img/011.png)

Log Buffer：日志缓冲区，用来保存要写入到磁盘中的log日志数据（redo log 、undo log）， 默认大小为 16MB，日志缓冲区的日志会定期刷新到磁盘中。如果需要更新、插入或删除许多行的事 务，增加日志缓冲区的大小可以节省磁盘 I/O。

######  参数

- `innodb_log_buffer_size`：缓冲区大小 

  ```mysql
  show variables like 'innodb_log_buffer_size';
  ```

- `innodb_flush_log_at_trx_commit`：日志刷新到磁盘时机

  ```mysql
  -- 取值主要包含以下三个：
  -- 默认值: 1: 日志在每次事务提交时写入并刷新到磁盘。
  -- 0: 每秒将日志写入并刷新到磁盘一次。
  -- 2: 日志在每次事务提交后写入，并每秒刷新到磁盘一次。
  show variables like 'innodb_flush_log_at_trx_commit';
  ```

#### 磁盘结构

##### System Tablespace/File-Per-Table Tablespaces

![](https://github.com/cbxbj/mysql/blob/master/img/012.png)

**System Tablespace**

系统表空间是更改缓冲区的存储区域。如果每一张表的独立表空间是关闭的，它也可能包含表和索引数据。(在MySQL5.x版本中还包含InnoDB数据字典、undolog等)

**File-Per-Table Tablespaces**

如果开启了innodb_file_per_table开关 ，则每个表的文件表空间包含单个InnoDB表的数据和索引 ，并存储在文件系统上的单个数据文件中。

###### 参数

`innodb_data_file_path`：系统表空间

```mysql
show variables like 'innodb_data_file_path';
```

系统表空间，默认的文件名叫 ibdata1。

`innodb_file_per_table`：数据库文件表空间

```mysql
show variables like 'innodb_file_per_table';
```

##### General Tablespaces/ Undo Tablespaces/Temporary Tablespaces

![](https://github.com/cbxbj/mysql/blob/master/img/013.png)

**General Tablespaces**

通用表空间，需要通过 CREATE TABLESPACE 语法创建通用表空间，在创建表时，可以指定该表空间。

###### 参数

```mysql
-- 1.创建表空间
CREATE TABLESPACE ts_name ADD DATAFILE 'file_name' ENGINE = engine_name;

-- 2.创建表时指定表空间
CREATE TABLE table_name TABLESPACE ts_name;
```

**Undo Tablespaces**

撤销表空间，MySQL实例在初始化时会自动创建两个默认的undo表空间（初始大小16M），用于存储 undo log日志。

默认名：`undo_001`、`undo_002`

 **Temporary Tablespaces**

临时表空间，InnoDB 使用会话临时表空间和全局临时表空间。存储用户创建的临时表等数据。

##### Doublewrite Buffer Files/Redo Log

![](https://github.com/cbxbj/mysql/blob/master/img/014.png)

**Doublewrite Buffer Files**

双写缓冲区，innoDB引擎将数据页从Buffer Pool刷新到磁盘前，先将数据页写入双写缓冲区文件中，便于系统异常时恢复数据。

默认名：`#ib_16384_0.dblwr`、`#ib_16384_1.dblwr`

**Redo Log**

重做日志，是用来实现事务的持久性。该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log）,前者是在内存中，后者在磁盘中。当事务提交之后会把所 有修改信息都会存到该日志中, 用于在刷新脏页到磁盘时,发生错误时, 进行数据恢复使用。

以循环方式写入重做日志文件，涉及两个文件：`ib_logfile0`、`ib_logfile1`

#### 后台线程

![](https://github.com/cbxbj/mysql/blob/master/img/015.png)

在InnoDB的后台线程中，分为4类，分别是：

- Master Thread 
- IO Thread
- Purge Thread
- Page Cleaner Thread

##### Master Thread

核心后台线程，负责调度其他线程，还负责将缓冲池中的数据异步刷新到磁盘中, 保持数据的一致性， 还包括脏页的刷新、合并插入缓存、undo页的回收 。

##### IO Thread

在InnoDB存储引擎中大量使用了AIO来处理IO请求, 这样可以极大地提高数据库的性能，而IO Thread主要负责这些IO请求的回调。

| 线程类型             | 默认个数 | 职责                         |
| -------------------- | -------- | ---------------------------- |
| Read thread          | 4        | 负责读操作                   |
| Write thread         | 4        | 负责写操作                   |
| Log thread           | 1        | 负责将日志缓冲区刷新到磁盘   |
| Insert buffer thread | 1        | 负责将写缓冲区内容刷新到磁盘 |

###### 参数

查看InnoDB的状态信息，其中就包含IO Thread信息

```mysql
show engine innodb status \G;
```

##### Purge Thread

主要用于回收事务已经提交了的undo log，在事务提交之后，undo log可能不用了，就用它来回收。

##### Page Cleaner Thread

协助 Master Thread 刷新脏页到磁盘的线程，它可以减轻 Master Thread 的工作压力，减少阻 塞。

### 事务原理

#### 事务基础

事务是一组操作的集合，它是一个不可分割的工作单位，事务会把所有的操作作为一个整体一起向系 统提交或撤销操作请求，即这些操作要么同时成功，要么同时失败。

##### 特性

- 原子性（Atomicity）：事务是不可分割的最小操作单元，要么全部成功，要么全部失败
-  一致性（Consistency）：事务完成时，必须使所有的数据都保持一致状态
- 隔离性（Isolation）：数据库系统提供的隔离机制，保证事务在不受外部并发操作影响的独立环 境下运行
- 持久性（Durability）：事务一旦提交或回滚，它对数据库中的数据的改变就是永久的。

![](https://github.com/cbxbj/mysql/blob/master/img/016.png)

#### redolog

重做日志，记录的是事务提交时数据页的物理修改，**是用来实现事务的持久性**。

![](https://github.com/cbxbj/mysql/blob/master/img/017.png)

该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log file）,前者是在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都存到该日志文件中, 用 于在刷新脏页到磁盘,发生错误时, 进行数据恢复使用。

##### 流程

- 先看Buffer Pool有没有相应数据，无则后台线程先从xxx.ibd文件中读取出来，再缓存到缓冲区
- 操作缓冲区的数据，该数据所在的页变为脏页
- 把增删改的数据记录在Redolog buffer中，(默认事务提交后，写入并刷新到磁盘)
- 通过一定的规则将缓冲区中脏页数据刷新到磁盘中
- **若刷新出错，通过磁盘中的redlog回滚**

每次刷新日志而非缓存数据原因：日志是顺序磁盘IO；缓存数据刷新到.ibd则很有可能是随机磁盘IO

因为在业务操作中，我们操作数据一般都是随机读写磁盘的，而不是顺序读写磁盘。 而redo log在 往磁盘文件中写入数据，由于是日志文件，所以都是顺序写的。顺序写的效率，要远大于随机写。 这 种先写日志的方式，称之为 WAL（Write-Ahead Logging）。

#### undolog

回滚日志，**用于记录数据被修改前的信息** , 作用包含两个 : 提供回滚(保证**事务的原子性**) 和 **MVCC**(多版本并发控制) 。

undo log和redo log记录物理日志不一样，它是逻辑日志。可以认为当delete一条记录时，undo log中会记录一条对应的insert记录，反之亦然，当update一条记录时，它记录一条对应相反的 update记录。当执行rollback时，就可以从undo log中的逻辑记录读取到相应的内容并进行回滚。

Undo log销毁：undo log在事务执行时产生，事务提交时，并不会立即删除undo log，因为这些 日志可能还用于MVCC。

Undo log存储：**undo log采用段的方式进行管理和记录**，存放在前面介绍的 rollback segment 回滚段中，内部包含1024个undo log segment。

#### MVCC

全称 `Multi-Version Concurrency Control`，多版本并发控制。指维护一个数据的多个版本， 使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能。MVCC的具体实现，还需要依赖于**数据库记录中的三个隐式字段、undo log日志、readView**。

##### 当前读

读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加 锁。对于我们日常的操作，如：select ... lock in share mode(共享锁)，select ... for update、update、insert、delete(排他锁)都是一种当前读。

###### 解析

`MySQL`默认隔离级别`Repeatable Read(可重复读)`，当前事务读取不到，开启事务后第一个select语句之后的最新数据；利用`select ... lock in share mode`语句可以读取到最新的数据

##### 快照读

简单的select（不加锁）就是快照读，快照读，读取的是记录数据的可见版本，有可能是历史数据， 不加锁，是非阻塞读。

- Read Committed：每次select，都生成一个快照读。 
- Repeatable Read：开启事务后第一个select语句才是快照读的地方。
- Serializable：快照读会退化为当前读。

##### 隐藏字段

![](https://github.com/cbxbj/mysql/blob/master/img/018.png)

| 隐藏字段    | 含义                                                         |
| ----------- | ------------------------------------------------------------ |
| DB_TRX_ID   | 最近修改事务ID，记录插入这条记录或最后一次修改该记录的事务ID。 |
| DB_ROLL_PTR | 回滚指针，指向这条记录的上一个版本，用于配合undo log，指向上一个版 本。 |
| DB_ROW_ID   | 隐藏主键，如果表结构没有指定主键，将会生成该隐藏字段。       |

查看表的结构可在cmd下使用命令

```mysql
ibd2sdi xxx.ibd
```

##### undolog

回滚日志，在insert、update、delete的时候产生的便于数据回滚的日志。

当insert的时候，产生的undo log日志只在回滚时需要，在事务提交后，可被立即删除。 **而update、delete的时候，产生的undo log日志不仅在回滚时需要，在快照读时也需要，不会立即被删除。**

###### 版本链

![](https://github.com/cbxbj/mysql/blob/master/img/019.png)

> 不同事务或相同事务对同一条记录进行修改，会导致该记录的undolog生成一条 记录版本链表，链表的头部是最新的旧记录，链表尾部是最早的旧记录

##### readview

ReadView（读视图）是 快照读 SQL执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务 （未提交的）id。

ReadView中包含了四个核心字段：

| 字段           | 含义                                                 |
| -------------- | ---------------------------------------------------- |
| m_ids          | 当前活跃的事务ID集合                                 |
| min_trx_id     | 最小活跃事务ID                                       |
| max_trx_id     | 预分配事务ID，当前最大事务ID+1（因为事务ID是自增的） |
| creator_trx_id | ReadView创建者的事务ID                               |

![](https://github.com/cbxbj/mysql/blob/master/img/020.png)

trx_id：代表当前事务id

| 条件                               | 是否可以访问                               | 说明                                      |
| ---------------------------------- | ------------------------------------------ | ----------------------------------------- |
| trx_id == creator_trx_id           | 可以访问该版本                             | 成立，说明数据是当前这个事 务更改的       |
| trx_id < min_trx_id                | 可以访问该版本                             | 成立，说明数据已经提交了                  |
| trx_id > max_trx_id                | 不可以访问该版本                           | 成立，说明该事务是在 ReadView生成后才开启 |
| min_trx_id <= trx_id <= max_trx_id | 如果trx_id不在m_ids中， 是可以访问该版本的 | 成立，说明数据已经提交                    |

不同的隔离级别，生成ReadView的时机不同

- READ COMMITTED ：在事务中每一次执行快照读时生成ReadView
- REPEATABLE READ：仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView

###### RC隔离级别

![](https://github.com/cbxbj/mysql/blob/master/img/021.png)

上图中

- 先匹配表中的数据这条记录对应的 trx_id为4，也就是将4带入右侧的匹配规则中。 ①不满足 ②不满足 ③不满足 ④也不满足 ， 都不满足，则继续匹配undo log版本链的下一条
- 再匹配第二条undolog中的数据，这条 记录对应的trx_id为3，也就是将3带入右侧的匹配规则中。①不满足 ②不满足 ③不满足 ④也 不满足 ，都不满足，则继续匹配undo log版本链的下一条
- 再匹配，这条记录对应的trx_id为2，也就是将2带入右侧的匹配规则中。①不满足 ②满足 终止匹配，此次快照 读，返回的数据就是版本链中记录的这条数据

![](https://github.com/cbxbj/mysql/blob/master/img/022.png)

上图中

- 先匹配表中的数据这条记录对应的 trx_id为4，也就是将4带入右侧的匹配规则中。 ①不满足 ②不满足 ③不满足 ④也不满足 ， 都不满足，则继续匹配undo log版本链的下一条
- 再匹配undolog中的数据，这条 记录对应的trx_id为3，也就是将3带入右侧的匹配规则中。①不满足 ②满足 。终止匹配，此次快照读，返回的数据就是版本链中记录的这条数据

###### RR隔离级别

RR隔离级别下，仅在事务中第一次执行快照读时生成ReadView，后续复用该ReadView。 而RR 是可 重复读，在一个事务中，执行两次相同的select语句，查询到的结果是一样的

![](https://github.com/cbxbj/mysql/blob/master/img/023.png)

#### 总结

![](https://github.com/cbxbj/mysql/blob/master/img/024.png)

## MySQL管理

### 系统数据库

Mysql数据库安装完成后，自带了一下四个数据库，具体作用如下

| 数据库             | 含义                                                         |
| ------------------ | ------------------------------------------------------------ |
| mysql              | 存储MySQL服务器正常运行所需要的各种信息 （时区、主从、用 户、权限等） |
| information_schema | 提供了访问数据库元数据的各种表和视图，包含数据库、表、字段类 型及访问权限等 |
| performance_schema | 为MySQL服务器运行时状态提供了一个底层监控功能，主要用于收集 数据库服务器性能参数 |
| sys                | 包含了一系列方便 DBA 和开发人员利用 performance_schema 性能数据库进行性能调优和诊断的视图 |

### 常用工具

以下命令均在cmd下输入

#### mysql

```mysql
语法 ：
	mysql [options] [database]
选项 ：
	-u, --user=name 		#指定用户名
	-p, --password[=name] 	#指定密码
	-h, --host=name 		#指定服务器IP或域名
	-P, --port=port 		#指定连接端口
	-e, --execute=name 		#执行SQL语句并退出
```

-e选项可以在Mysql客户端执行SQL语句，而不用连接到MySQL数据库再执行，对于一些批处理脚本， 这种方式尤其方便。

示例：

```mysql
mysql -uroot –p123456 db01 -e "select * from test";
```

#### mysqladmin

`mysqladmin` 是一个执行管理操作的客户端程序。可以用它来检查服务器的配置和当前状态、创建并 删除数据库等

```mysql
语法:
	mysqladmin [options] command ...
选项:
	-u, --user=name 		#指定用户名
	-p, --password[=name] 	#指定密码
	-h, --host=name 		#指定服务器IP或域名
	-P, --port=port			#指定连接端口
```

示例：

```mysql
mysqladmin -uroot –p123456 drop 'test';
mysqladmin -uroot –p123456 version;
```

#### mysqlbinlog

由于服务器生成的二进制日志文件以二进制格式保存，所以如果想要检查这些文本的文本格式，就会使 用到mysqlbinlog 日志管理工具。

```mysql
语法 ：
	mysqlbinlog [options] log-files1 log-files2 ...
选项 ：
    -d, --database=name 							#指定数据库名称，只列出指定的数据库相关操作。
    -o, --offset=									#忽略掉日志中的前n行命令。
    -r,--result-file=name 							#将输出的文本格式日志输出到指定文件。
    -s, --short-form 								#显示简单格式， 省略掉一些信息。
    --start-datatime=date1 --stop-datetime=date2 	#指定日期间隔内的所有日志。
    --start-position=pos1 --stop-position=pos2 		#指定位置间隔内的所有日志。
```

示例:

```mysql
mysqlbinlog -s binlog.000001
```

#### mysqlshow

mysqlshow 客户端对象查找工具，用来很快地查找存在哪些数据库、数据库中的表、表中的列或者索引。

```mysql
语法 ：
	mysqlshow [options] [db_name [table_name [col_name]]]
选项 ：
    --count #显示数据库及表的统计信息（数据库，表 均可以不指定）
    -i 		#显示指定数据库或者指定表的状态信息
示例：
#查询test库中每个表中的字段书，及行数
mysqlshow -uroot -p123456 test --count
#查询test库中book表的详细情况
mysqlshow -uroot -p123456 test book --count
```

#### mysqldump

mysqldump 客户端工具用来备份数据库或在不同数据库之间进行数据迁移。备份内容包含创建表，及 插入表的SQL语句。

```mysql
语法 ：
    mysqldump [options] db_name [tables]
    mysqldump [options] --database/-B db1 [db2 db3...]
    mysqldump [options] --all-databases/-A
连接选项 ：
    -u, --user=name 		#指定用户名
    -p, --password[=name] 	#指定密码
    -h, --host=name 		#指定服务器ip或域名
    -P, --port=				#指定连接端口
输出选项：
    --add-drop-database 	#在每个数据库创建语句前加上 drop database 语句
    --add-drop-table 		#在每个表创建语句前加上 drop table 语句 , 默认开启 ; 不开启 (--skip-add-drop-table)
    -n, --no-create-db 		#不包含数据库的创建语句
    -t, --no-create-info 	#不包含数据表的创建语句
    -d, --no-data 			#不包含数据
    -T, --tab=name 			#自动生成两个文件：一个.sql文件，创建表结构的语句；一个.txt文件，数据文件
```

示例：

```mysql
mysqldump -uroot -p123456 db01 > db01.sql
```

注：-T参数中生成的.txt文件必须在MySQL信任的目录下，

查询目录语句`SHOW VARIABLES LIKE '%secure_file_priv%'`

```mysql
mysqldump -uroot -p123456 -T C:\ProgramData\MySQL\MySQL Server 8.0\Uploads\ db01 score
```

#### mysqlimport

 mysqlimport mysqlimport 是客户端数据导入工具，用来导入mysqldump 加 -T 参数后导出的文本文件

```mysql
语法 ：
	mysqlimport [options] db_name textfile1 [textfile2...]
示例 ：
mysqlimport -uroot -p123456 db01 C:\ProgramData\MySQL\MySQL Server 8.0\Uploads\score.txt
```

#### source

source 如果需要导入sql文件,可以使用mysql中的source 指令

```mysql
-- 注：在mysqln内输入，语法 ：
	source C:\score.sql
```

