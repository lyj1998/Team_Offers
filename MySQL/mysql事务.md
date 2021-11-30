## mysql事务

### 1、什么是事务

事务是一个不可分割的数据库操作序列，也是数据库并发控制的基本单位，其执行的结果必须使数据库从一种一致性状态变到另一种一致性状态。事务是逻辑上的一组操作，要么都执行，要么都不执行。



### 2、事务的四大特性（ACID）

**原子性**： 事务是最小的执行单位，不允许分割。事务的原子性确保动作要么全部完成，要么完全不起作用；
**一致性**： 执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的；
**隔离**性： 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的；
**持久性**： 一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

### 3、脏读、幻读、不可重复读

**脏读(Drity Read)**：某个事务已更新一份数据，另一个事务在此时读取了同一份数据，由于某些原因，前一个RollBack了操作，则后一个事务所读取的数据就会是不正确的。
**不可重复读(Non-repeatable read)**:在一个事务的两次查询之中数据不一致，这可能是两次查询过程中间插入了一个事务更新的原有的数据。
**幻读(Phantom Read)**:在一个事务的两次查询中数据笔数不一致，例如有一个事务查询了几列(Row)数据，而另一个事务却在此时插入了新的几列数据，先前的事务在接下来的查询中，就会发现有几列数据是它先前所没有的。



### 4、事务的隔离级别



![截屏2021-11-29 下午5.24.11](/Users/yangtao303/Library/Application Support/typora-user-images/截屏2021-11-29 下午5.24.11.png)

**READ-UNCOMMITTED(读取未提交)：** 最低的隔离级别，允许读取尚未提交的数据变更，可能会导致脏读、幻读或不可重复读。
**READ-COMMITTED(读取已提交)：** 允许读取并发事务已经提交的数据，可以阻止脏读，但是幻读或不可重复读仍有可能发生。
**REPEATABLE-READ(可重复读)：** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，可以阻止脏读和不可重复读，但幻读仍有可能发生。
**SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。

Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别



### 5、例子（我认为是重点）具体去解释事务的隔离级别以及问题的解决方法

#### 首先隔离级别是读未提交，这个时候是发生三个问题的，

设置mysql隔离级别的语句：

```mysql
//设置read uncommitted级别：
set session transaction isolation level read uncommitted;

//设置read committed级别：
set session transaction isolation level read committed;

//设置repeatable read级别：
set session transaction isolation level repeatable read;

//设置serializable级别：
set session transaction isolation level serializable;
```



首先设置为读未提交的隔离级别：



1、读未提交：

　　　　（1）打开一个客户端A，并设置当前事务模式为read uncommitted（未提交读），查询表account的初始值：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615225939087-367776221.png)

　（2）在客户端A的事务提交之前，打开另一个客户端B，更新表account：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615230218306-862399438.png)

　（3）这时，虽然客户端B的事务还没提交，但是客户端A就可以查询到B已经更新的数据：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615230427790-2059251412.png)

（4）一旦客户端B的事务因为某种原因回滚，所有的操作都将会被撤销，那客户端A查询到的数据其实就是脏数据：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615230655400-1018252120.png)

（5）在客户端A执行更新语句update account set balance = balance - 50 where id =1，lilei的balance没有变成350，居然是400，是不是很奇怪，数据的一致性没问啊，如果你这么想就太天真 了，在应用程序中，我们会用400-50=350，并不知道其他会话回滚了，要想解决这个问题可以采用读已提交的隔离级别

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170616203815181-1023048699.png)

#### 

#### 接着是读已提交，这个隔离级别解决了脏读问题，需要去解决不可重复读问题

2、读已提交

　　　　（1）打开一个客户端A，并设置当前事务模式为read committed（未提交读），查询表account的初始值：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615231437353-1441361659.png)

（2）在客户端A的事务提交之前，打开另一个客户端B，更新表account：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615231920696-48081094.png)

（3）这时，客户端B的事务还没提交，客户端A不能查询到B已经更新的数据，解决了脏读问题：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615232203978-179631977.png)

（4）客户端B的事务提交

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615232506650-1677223761.png)



（5）客户端A执行与上一步相同的查询，结果 与上一步不一致，即产生了不可重复读的问题，在应用程序中，假设我们处于客户端A的会话，查询到lilei的balance为450，但是其他事务将lilei的balance值改为400，我们并不知道，如果用450这个值去做其他操作，是有问题的，不过这个概率真的很小哦，要想避免这个问题，可以采用可重复读的隔离级别

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615232748337-2092924598.png)



#### 

#### 再接着是可重复读隔离级别，解决了不可重复读问题，需要去解决幻读问题

3、可重复读

 　　　（1）打开一个客户端A，并设置当前事务模式为repeatable read，查询表account的初始值：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615233320290-1840487787.png)

（2）在客户端A的事务提交之前，打开另一个客户端B，更新表account并提交，客户端B的事务居然可以修改客户端A事务查询到的行，也就是mysql的可重复读不会锁住事务查询到的行，这一点出乎我的意料，sql标准中事务隔离级别为可重复读时，读写操作要锁行的，mysql居然没有锁，我了个去。在应用程序中要注意给行加锁，不然你会以步骤（1）中lilei的balance为400作为中间值去做其他操作

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615233526103-1495989601.png)

（3）在客户端A执行步骤（1）的查询：

![img](http://images2015.cnblogs.com/blog/1183794/201706/1183794-20170615233858087-1000794949.png)



4）执行步骤（1），lilei的balance仍然是400与步骤（1）查询结果一致，没有出现不可重复读的 问题；接着执行update balance = balance - 50 where id = 1，balance没有变成400-50=350，lilei的balance值用的是步骤（2）中的350来算的，所以是300，数据的一致性倒是没有被破坏，这个有点神奇，也许是mysql的特色吧



```
mysql> select * from account;
+------+--------+---------+
| id   | name   | balance |
+------+--------+---------+
|    1 | lilei  |     400 |
|    2 | hanmei |   16000 |
|    3 | lucy   |    2400 |
+------+--------+---------+
3 rows in set (0.00 sec)


mysql> update account set balance = balance - 50 where id = 1;
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0


mysql> select * from account;
+------+--------+---------+
| id   | name   | balance |
+------+--------+---------+
|    1 | lilei  |     300 |
|    2 | hanmei |   16000 |
|    3 | lucy   |    2400 |
+------+--------+---------+
3 rows in set (0.00 sec) 
 
 
```

　　　　(5) 在客户端A开启事务，查询表account的初始值

```
 mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)


mysql> select * from account;
+------+--------+---------+
| id | name | balance |
+------+--------+---------+
| 1 | lilei | 300 |
| 2 | hanmei | 16000 |
| 3 | lucy | 2400 |
+------+--------+---------+
3 rows in set (0.00 sec) 



```

　　　　（6）在客户端B开启事务，新增一条数据，其中balance字段值为600，并提交

```
mysql> start transaction;
Query OK, 0 rows affected (0.00 sec) mysql> insert into account values(4,'lily',600); Query OK, 1 row affected (0.00 sec) mysql> commit; Query OK, 0 rows affected (0.01 sec)
```

　　　　(7) 在客户端A计算balance之和，值为300+16000+2400=18700，没有把客户端B的值算进去，客户端A提交后再计算balance之和，居然变成了19300，这是因为把客户端B的600算进去了

，站在客户的角度，客户是看不到客户端B的，它会觉得是天下掉馅饼了，多了600块，这就是幻读，站在开发者的角度，数据的 一致性并没有破坏。但是在应用程序中，我们得代码可能会把18700提交给用户了，如果你一定要避免这情况小概率状况的发生，那么就要采取下面要介绍的事务隔离级别“串行化”

mysql> select sum(balance) from account;
+--------------+
| sum(balance) |
+--------------+
| 18700 |
+--------------+
1 row in set (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)

mysql> select sum(balance) from account;
+--------------+
| sum(balance) |
+--------------+
| 19300 |
+--------------+
1 row in set (0.00 sec)



#### 最后是可串行化的隔离级别，解决了所有的问题

　　4.串行化

　　　　（1）打开一个客户端A，并设置当前事务模式为serializable，查询表account的初始值：

```
mysql> set session transaction isolation level serializable;
Query OK, 0 rows affected (0.00 sec)


mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)


mysql> select * from account;
+------+--------+---------+
| id   | name   | balance |
+------+--------+---------+
|    1 | lilei  |   10000 |
|    2 | hanmei |   10000 |
|    3 | lucy   |   10000 |
|    4 | lily   |   10000 |
+------+--------+---------+
4 rows in set (0.00 sec)

```

　　　　（2）打开一个客户端B，并设置当前事务模式为serializable，插入一条记录报错，表被锁了插入失败，mysql中事务隔离级别为serializable时会锁表，因此不会出现幻读的情况，这种隔离级别并发性极低，往往一个事务霸占了一张表，其他成千上万个事务只有干瞪眼，得等他用完提交才可以使用，开发中很少会用到。

```
mysql> set session transaction isolation level serializable;
Query OK, 0 rows affected (0.00 sec)

mysql> start transaction;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into account values(5,'tom',0);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```



### 6、MVCC，其实也是不可重复读的解决方法的实现原理

#### 1、什么是MVCC

​         **`MVCC`**，全称 `Multi-Version Concurrency Control` ，即**多版本并发控制**。MVCC 是一种并发控制的方法，一般在数据库管理系统中，实现对数据库的并发访问，在编程语言中实现事务内存。

​     **MVCC** 在 **MySQL InnoDB** 中的实现主要是为了提高数据库并发性能，用更好的方式去处理读-写冲突，做到即使有读写冲突时，也能做到不加锁，非阻塞并发读

#### 2、什么是快照读和当前读？

​         **当前读：**

        像 select lock in share mode (共享锁), select for update; update; insert; delete (排他锁)这些操作都是一种当前读，为什么叫当前读？就是它读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对读取的记录进行加锁
​           

​          **快照读：**

          像不加锁的 select 操作就是快照读，即不加锁的非阻塞读；快照读的前提是隔离级别不是串行级别，串行级别下的快照读会退化成当前读；之所以出现快照读的情况，是基于提高并发性能的考虑，快照读的实现是基于多版本并发控制，即 MVCC ,可以认为 MVCC 是行锁的一个变种，但它在很多情况下，避免了加锁操作，降低了开销；既然是基于多版本，即快照读可能读到的并不一定是数据的最新版本，而有可能是之前的历史版本


#### 3、当前读，快照读和MVCC的关系

​       MVCC 多版本并发控制是 「维持一个数据的多个版本，使得读写操作没有冲突」 的概念，只是一个抽象概念，并非实现
因为 MVCC 只是一个抽象概念，要实现这么一个概念，MySQL 就需要提供具体的功能去实现它，「快照读就是 MySQL 实现 MVCC 理想模型的其中一个非阻塞读功能」。而相对而言，当前读就是悲观锁的具体功能实现
要说的再细致一些，快照读本身也是一个抽象概念，再深入研究。MVCC 模型在 MySQL 中的具体实现则是由 3 个隐式字段，undo 日志 ，Read View 等去完成的，具体可以看下面的 MVCC 实现原理



#### 4、MVCC 能解决什么问题，好处是？

                  多版本并发控制（MVCC）是一种用来解决读-写冲突的无锁并发控制，也就是为事务分配单向增长的时间戳，为每个修改保存一个版本，版本与事务时间戳关联，读操作只读该事务开始前的数据库的快照。 所以 MVCC 可以为数据库解决以下问题

在并发读写数据库时，可以做到在读操作时不用阻塞写操作，写操作也不用阻塞读操作，提高了数据库并发读写的性能
同时还可以解决脏读，幻读，不可重复读等事务隔离问题，但不能解决更新丢失问题

**在数据库中，因为有了 MVCC，所以我们可以形成两个组合：**

- `MVCC + 悲观锁`
  MVCC解决读写冲突，悲观锁解决写写冲突
- `MVCC + 乐观锁`
  MVCC 解决读写冲突，乐观锁解决写写冲突



#### 5、MVCC的实现原理

MVCC 的目的就是多版本并发控制，在数据库中的实现，就是为了解决`读写冲突`，它的实现原理主要是依赖记录中的 **`3个隐式字段`**，**`undo日志`** ，**`Read View`** 来实现的。

#####        

##### **隐式字段**

DB_TRX_ID：6 byte，最近修改(`修改/插入`)事务 ID：记录创建这条记录/最后一次修改该记录的事务 ID

DB_ROLL_PTR：7 byte，回滚指针，指向这条记录的上一个版本（存储于 rollback segment 里）

DB_ROW_ID：6 byte，隐含的自增 ID（隐藏主键），如果数据表没有主键，InnoDB 会自动以`DB_ROW_ID`产生一个聚簇索引

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190313213705258.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)

Id name money zhujianID shiwuID 指针

##### **undo日志**

insert undo log
代表事务在 insert 新记录时产生的 undo log, 只在事务回滚时需要，并且在事务提交后可以被立即丢弃
update undo log
事务在进行 update 或 delete 时产生的 undo log ; 不仅在事务回滚时需要，在快照读时也需要；所以不能随便删除，只有在快速读或事务回滚不涉及该日志时，对应的日志才会被 purge 线程统一清除

**其实就是记录链**

![img](https://img-blog.csdnimg.cn/20190313220528630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NuYWlsTWFubg==,size_16,color_FFFFFF,t_70)



##### **Read View 读视图**

        什么是 Read View，说白了 Read View 就是事务进行快照读操作的时候生产的读视图 (Read View)，在该事务执行的快照读的那一刻，会生成数据库系统当前的一个快照，记录并维护系统当前活跃事务的 ID (当每个事务开启时，都会被分配一个 ID , 这个 ID 是递增的，所以最新的事务，ID 值越大)

所以我们知道 Read View 主要是用来做可见性判断的, 即当我们某个事务执行快照读的时候，对该记录创建一个 Read View 读视图，把它比作条件用来判断当前事务能够看到哪个版本的数据，既可能是当前最新的数据，也有可能是该行记录的undo log里面的某个版本的数据。


## InnoDB解决幻读的方法

- 多版本并发控制（MVCC，快照读）
- next-key锁（当前读）

### MVCC

mvcc（Multi-Version Concurrency Control）多版本并发控制实现，通过保存数据在某个时间点的快照来实现的。一个事务无论运行多长时间，在同一个事务里能够看到数据一致的视图。根据事务开始的时间不同，同时也意味着在同一个时刻不同事务看到的相同表里的数据可能是不同的。

在每一行数据中额外保存两个隐藏的列：当前行创建时的版本号和删除时的版本号（可能为空，其实还有一列称为回滚指针，用于事务回滚，不在本文范畴）。这里的版本号并不是实际的时间值，而是系统版本号。每开始新的事务，系统版本号都会自动递增。事务开始时刻的系统版本号会作为事务的版本号，用来和查询每行记录的版本号进行比较。

每个事务又有自己的版本号，这样事务内执行CRUD操作时，就通过版本号的比较来达到数据版本控制的目的。

### next-key 

InnoDB是一个支持行锁的存储引擎，有三种排它锁（X锁）：

- Record Lock：行锁，单个行记录上的锁。
- Gap Lock：间隙锁，锁定一个范围，但不包括记录本身。GAP锁的目的，是为了防止幻读、防止间隙内有新数据插入、防止已存在的数据更新为间隙内的数据。
- Next-Key Lock：1+2，锁定一个范围，并且锁定记录本身。对于行的查询，都是采用该方法，主要目的是解决幻读的问题。InnoDB默认加锁方式是next-key 锁。

next-key锁是mysql默认的锁，是记录锁和在此索引记录之前的gap上的锁的结合，这个锁的作用是为了防止幻读，导致主从复制的不一致。

比如有一个表，id列上有90,100,102。

当我们执行select * from order where id=100 for update 时，mysql会锁住90到102这个区间。当我们执行 select * from order where id>100 for update时，这时next-key锁就派上用场了。

索引扫描到了100和102这两个值，但是仅仅锁住这两个值是不够的，因为当我们在另一个会话插入id=101的时候，就有可能产生幻读了。所以mysql必须锁住[100,102)和[102,无穷大）这个范围，才能保证不会出现幻读。

