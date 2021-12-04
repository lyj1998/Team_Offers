## mysql优化

### 1、SQL优化

#### 1、索引优化（避免不走索引的场景）：

##### **尽量避免在字段开头模糊查询，会导致数据库引擎放弃索引进行全表扫描。**

SELECT * FROM t WHERE username LIKE '%陈%'
优化方式：尽量在字段后面使用模糊查询。如下：

SELECT * FROM t WHERE username LIKE '陈%'
如果需求是要在前面使用模糊查询，

使用MySQL内置函数INSTR(str,substr) 来匹配，作用类似于java中的indexOf()，查询字符串出现的角标位置，可参阅《MySQL模糊查询用法大全（正则、通配符、内置函数等）》
使用FullText全文索引，用match against 检索
数据量较大的情况，建议引用ElasticSearch、solr，亿级数据量检索速度秒级
当表数据量较少（几千条儿那种），别整花里胡哨的，直接用like '%xx%'。

##### **尽量避免使用in 和not in，会导致引擎走全表扫描。**

SELECT * FROM t WHERE id IN (2,3)
优化方式：如果是连续数值，可以用between代替。如下：

SELECT * FROM t WHERE id BETWEEN 2 AND 3
如果是子查询，可以用exists代替。详情见《MySql中如何用exists代替in》如下：

-- 不走索引
select * from A where A.id in (select id from B);
-- 走索引
select * from A where exists (select * from B where B.id = A.id);

##### ** 尽量避免使用 or，会导致数据库引擎放弃索引进行全表扫描。**

SELECT * FROM t WHERE id = 1 OR id = 3
优化方式：可以用union代替or。如下：

SELECT * FROM t WHERE id = 1
   UNION
SELECT * FROM t WHERE id = 3
##### 尽量避免进行null值的判断，会导致数据库引擎放弃索引进行全表扫描。如下：

SELECT * FROM t WHERE score IS NULL
优化方式：可以给字段添加默认值0，对0值进行判断。如下：

SELECT * FROM t WHERE score = 0

##### **尽量避免在where条件中等号的左侧进行表达式、函数操作，会导致数据库引擎放弃索引进行全表扫描。**

可以将表达式、函数操作移动到等号右侧。如下：

```sql
-- 全表扫描
SELECT * FROM T WHERE score/10 = 9

-- 走索引
SELECT * FROM T WHERE score = 10*9
```

##### ** 查询条件不能用 <> 或者 !=**

使用索引列作为条件进行查询时，需要避免使用<>或者!=等判断条件。如确实业务需要，使用到不等于符号，需要在重新评估索引建立，避免在此字段上建立索引，改由查询条件中其他索引字段代替。

##### **where条件仅包含复合索引非前置列**

如下：复合（联合）索引包含key_part1，key_part2，key_part3三列，但SQL语句没有包含索引前置列"key_part1"，按照MySQL联合索引的最左匹配原则，不会走联合索引。详情参考《联合索引的使用原理》。

select col1 from table where key_part2=1 and key_part3=2



##### 隐式类型转换造成不使用索引** 

如下SQL语句由于索引对列类型为varchar，但给定的值为数值，涉及隐式类型转换，造成不能正确走索引。 

```sql
select col1 from table where col_varchar=123; 
```

#####  order by 条件要与where中条件一致，否则order by不会利用索引进行排序**

```sql
-- 不走age索引
SELECT * FROM t order by age;
 
-- 走age索引
SELECT * FROM t where age > 0 order by age;
```

对于上面的语句，数据库的处理顺序是：

第一步：根据where条件和统计信息生成执行计划，得到数据。
第二步：将得到的数据排序。当执行处理数据（order by）时，数据库会先查看第一步的执行计划，看order by 的字段是否在执行计划中利用了索引。如果是，则可以利用索引顺序而直接取得已经排好序的数据。如果不是，则重新进行排序操作。
第三步：返回排序后的数据。
当order by 中的字段出现在where条件中时，才会利用索引而不再二次排序，更准确的说，order by 中的字段在执行计划中利用了索引时，不用排序操作。

这个结论不仅对order by有效，对其他需要排序的操作也有效。比如group by 、union 、distinct等。


##### **正确使用hint优化语句**

MySQL中可以使用hint指定优化器在执行时选择或忽略特定的索引。一般而言，处于版本变更带来的表结构索引变化，更建议避免使用hint，而是通过Analyze table多收集统计信息。但在特定场合下，指定hint可以排除其他索引干扰而指定更优的执行计划。

USE INDEX 在你查询语句中表名的后面，添加 USE INDEX 来提供希望 MySQL 去参考的索引列表，就可以让 MySQL 不再考虑其他可用的索引。例子: SELECT col1 FROM table USE INDEX (mod_time, name)...
IGNORE INDEX 如果只是单纯的想让 MySQL 忽略一个或者多个索引，可以使用 IGNORE INDEX 作为 Hint。例子: SELECT col1 FROM table IGNORE INDEX (priority) ...
FORCE INDEX 为强制 MySQL 使用一个特定的索引，可在查询中使用FORCE INDEX 作为Hint。例子: SELECT col1 FROM table FORCE INDEX (mod_time) ...



#### 2、SELECT语句其他优化

##### **避免出现select \***

首先，select * 操作在任何类型数据库中都不是一个好的SQL编写习惯。

使用select * 取出全部列，会让优化器无法完成索引覆盖扫描这类优化，会影响优化器对执行计划的选择，也会增加网络带宽消耗，更会带来额外的I/O,内存和CPU消耗。

建议提出业务实际需要的列数，将指定列名以取代select *。具体详情见《为什么大家都说SELECT * 效率低》：



##### **避免出现不确定结果的函数**

特定针对主从复制这类业务场景。由于原理上从库复制的是主库执行的语句，使用如now()、rand()、sysdate()、current_user()等不确定结果的函数很容易导致主库与从库相应的数据不一致。另外不确定值的函数,产生的SQL语句无法利用query cache。



##### **多表关联查询时，小表在前，大表在后。**

在MySQL中，执行 from 后的表关联查询是从左往右执行的（Oracle相反），第一张表会涉及到全表扫描，所以将小表放在前面，先扫小表，扫描快效率较高，在扫描后面的大表，或许只扫描大表的前100行就符合返回条件并return了。

例如：表1有50条数据，表2有30亿条数据；如果全表扫描表2，你品，那就先去吃个饭再说吧是吧。



##### **使用表的别名**	

当在SQL语句中连接多个表时，请使用表的别名并把别名前缀于每个列名上。这样就可以减少解析的时间并减少哪些友列名歧义引起的语法错误。



##### **用where字句替换HAVING字句**

避免使用HAVING字句，因为HAVING只会在检索出所有记录之后才对结果集进行过滤，而where则是在聚合前刷选记录，如果能通过where字句限制记录的数目，那就能减少这方面的开销。HAVING中的条件一般用于聚合函数的过滤，除此之外，应该将条件写在where字句中。

where和having的区别：where后面不能使用组函数



##### **调整Where字句中的连接顺序**

MySQL采用从左往右，自上而下的顺序解析where子句。根据这个原理，应将过滤数据多的条件往前放，最快速度缩小结果集。



#### **3、增删改 DML 语句优化**

##### **大批量插入数据**

如果同时执行大量的插入，建议使用多个值的INSERT语句(方法二)。这比使用分开INSERT语句快（方法一），一般情况下批量插入效率有几倍的差别。

方法一：

insert into T values(1,2); 

insert into T values(1,3); 

insert into T values(1,4);
方法二：

Insert into T values(1,2),(1,3),(1,4); 

选择后一种方法的原因有三。 

减少SQL语句解析的操作，MySQL没有类似Oracle的share pool，采用方法二，只需要解析一次就能进行数据的插入操作；
在特定场景可以减少对DB连接次数
SQL语句较短，可以减少网络传输的IO。

##### **适当使用commit**

适当使用commit可以释放事务占用的资源而减少消耗，commit后能释放的资源如下：

- 事务占用的undo数据块；
- 事务在redo log中记录的数据块； 
- 释放事务施加的，减少锁争用影响性能。特别是在需要使用delete删除大量数据的时候，必须分解删除量并定期commit。



#####  避免重复查询更新的数据

针对业务中经常出现的更新行同时又希望获得改行信息的需求，MySQL并不支持PostgreSQL那样的UPDATE RETURNING语法，在MySQL中可以通过变量实现。

例如，更新一行记录的时间戳，同时希望查询当前记录中存放的时间戳是什么，简单方法实现：

Update t1 set time=now() where col1=1; 

Select time from t1 where id =1; 
使用变量，可以重写为以下方式： 

Update t1 set time=now () where col1=1 and @now: = now (); 

Select @now; 
前后二者都需要两次网络来回，但使用变量避免了再次访问数据表，特别是当t1表数据量较大时，后者比前者快很多。



##### 查询优先还是更新（insert、update、delete）优先

MySQL 还允许改变语句调度的优先级，它可以使来自多个客户端的查询更好地协作，这样单个客户端就不会由于锁定而等待很长时间。改变优先级还可以确保特定类型的查询被处理得更快。我们首先应该确定应用的类型，判断应用是以查询为主还是以更新为主的，是确保查询效率还是确保更新的效率，决定是查询优先还是更新优先。下面我们提到的改变调度策略的方法主要是针对只存在表锁的存储引擎，比如 MyISAM 、MEMROY、MERGE，对于Innodb 存储引擎，语句的执行是由获得行锁的顺序决定的。MySQL 的默认的调度策略可用总结如下：

1）写入操作优先于读取操作。

2）对某张数据表的写入操作某一时刻只能发生一次，写入请求按照它们到达的次序来处理。

3）对某张数据表的多个读取操作可以同时地进行。MySQL 提供了几个语句调节符，允许你修改它的调度策略：

LOW_PRIORITY关键字应用于DELETE、INSERT、LOAD DATA、REPLACE和UPDATE；
HIGH_PRIORITY关键字应用于SELECT和INSERT语句；
DELAYED关键字应用于INSERT和REPLACE语句。

       如果写入操作是一个 LOW_PRIORITY（低优先级）请求，那么系统就不会认为它的优先级高于读取操作。在这种情况下，如果写入者在等待的时候，第二个读取者到达了，那么就允许第二个读取者插到写入者之前。只有在没有其它的读取者的时候，才允许写入者开始操作。这种调度修改可能存在 LOW_PRIORITY写入操作永远被阻塞的情况。

SELECT 查询的HIGH_PRIORITY（高优先级）关键字也类似。它允许SELECT 插入正在等待的写入操作之前，即使在正常情况下写入操作的优先级更高。另外一种影响是，高优先级的 SELECT 在正常的 SELECT 语句之前执行，因为这些语句会被写入操作阻塞。如果希望所有支持LOW_PRIORITY 选项的语句都默认地按照低优先级来处理，那么 请使用--low-priority-updates 选项来启动服务器。通过使用 INSERTHIGH_PRIORITY 来把 INSERT 语句提高到正常的写入优先级，可以消除该选项对单个INSERT语句的影响。



#### 4、查询条件优化

##### **对于复杂的查询，可以使用中间临时表 暂存数据；**



##### ** 优化group by语句**

默认情况下，MySQL 会对GROUP BY分组的所有值进行排序，如 “GROUP BY col1，col2，....;” 查询的方法如同在查询中指定 “ORDER BY col1，col2，...;” 如果显式包括一个包含相同的列的 ORDER BY子句，MySQL 可以毫不减速地对它进行优化，尽管仍然进行排序。

因此，如果查询包括 GROUP BY 但你并不想对分组的值进行排序，你可以指定 ORDER BY NULL禁止排序。例如：

SELECT col1, col2, COUNT(*) FROM table GROUP BY col1, col2 ORDER BY NULL ;



##### **优化join语句**

MySQL中可以通过子查询来使用 SELECT 语句来创建一个单列的查询结果，然后把这个结果作为过滤条件用在另一个查询中。使用子查询可以一次性的完成很多逻辑上需要多个步骤才能完成的 SQL 操作，同时也可以避免事务或者表锁死，并且写起来也很容易。但是，有些情况下，子查询可以被更有效率的连接(JOIN)..替代。


例子：假设要将所有没有订单记录的用户取出来，可以用下面这个查询完成：

SELECT col1 FROM customerinfo WHERE CustomerID NOT in (SELECT CustomerID FROM salesinfo )

如果使用连接(JOIN).. 来完成这个查询工作，速度将会有所提升。尤其是当 salesinfo表中对 CustomerID 建有索引的话，性能将会更好，查询如下：

SELECT col1 FROM customerinfo 
   LEFT JOIN salesinfoON customerinfo.CustomerID=salesinfo.CustomerID 
      WHERE salesinfo.CustomerID IS NULL 
连接(JOIN).. 之所以更有效率一些，是因为 MySQL 不需要在内存中创建临时表来完成这个逻辑上的需要两个步骤的查询工作。



##### **优化union查询**

MySQL通过创建并填充临时表的方式来执行union查询。除非确实要消除重复的行，否则建议使用union all。原因在于如果没有all这个关键词，MySQL会给临时表加上distinct选项，这会导致对整个临时表的数据做唯一性校验，这样做的消耗相当高。

高效：

SELECT COL1, COL2, COL3 FROM TABLE WHERE COL1 = 10 

UNION ALL 

SELECT COL1, COL2, COL3 FROM TABLE WHERE COL3= 'TEST'; 
低效：

SELECT COL1, COL2, COL3 FROM TABLE WHERE COL1 = 10 

UNION 

SELECT COL1, COL2, COL3 FROM TABLE WHERE COL3= 'TEST';



##### **拆分复杂SQL为多个小SQL，避免大事务**

- 简单的SQL容易使用到MySQL的QUERY CACHE； 
- 减少锁表时间特别是使用MyISAM存储引擎的表； 
- 可以使用多核CPU。



##### **使用truncate代替delete**

当删除全表中记录时，使用delete语句的操作会被记录到undo块中，删除记录也记录binlog，当确认需要删除全表时，会产生很大量的binlog并占用大量的undo数据块，此时既没有很好的效率也占用了大量的资源。

使用truncate替代，不会记录可恢复的信息，数据不能被恢复。也因此使用truncate操作有其极少的资源占用与极快的时间。另外，使用truncate可以回收表的水位，使自增字段值归零。


##### **使用合理的分页方式以提高分页效率**

使用合理的分页方式以提高分页效率 针对展现等分页需求，合适的分页方式能够提高分页的效率。

案例1： 

select * from t where thread_id = 10000 and deleted = 0 
   order by gmt_create asc limit 0, 15;

       上述例子通过一次性根据过滤条件取出所有字段进行排序返回。数据访问开销=索引IO+索引全部记录结果对应的表数据IO。因此，该种写法越翻到后面执行效率越差，时间越长，尤其表数据量很大的时候。

适用场景：当中间结果集很小（10000行以下）或者查询条件复杂（指涉及多个不同查询字段或者多表连接）时适用。


案例2： 

select t.* from (select id from t where thread_id = 10000 and deleted = 0
   order by gmt_create asc limit 0, 15) a, t 
      where a.id = t.id; 
上述例子必须满足t表主键是id列，且有覆盖索引secondary key:(thread_id, deleted, gmt_create)。通过先根据过滤条件利用覆盖索引取出主键id进行排序，再进行join操作取出其他字段。数据访问开销=索引IO+索引分页后结果（例子中是15行）对应的表数据IO。因此，该写法每次翻页消耗的资源和时间都基本相同，就像翻第一页一样。

适用场景：当查询和排序字段（即where子句和order by子句涉及的字段）有对应覆盖索引时，且中间结果集很大的情况时适用。



### 2、主从复制与读写分离

mysql的主从复制读写分离，以及redis的主从复制哨兵模式都是保证系统的高可用性



##### 1、主从复制与读写分离的意义

企业中的业务通常数据量都比较大，而单台数据库在数据存储、安全性和高并发方面都无法满足实际的需求，所以需要配置多台主从数据服务器，以实现主从复制，增加数据可靠性，读写分离，也减少数据库压力和存储引擎带来的表锁定和行锁定问题。



##### 2、主从数据库实现同步（主从复制）

什么是主从复制？简单来说就是在主服务器上执行的语句，从服务器执行同样的语句，在主服务器上的操作在从服务器产生了同样的结果。

主从复制的基本过程如下:

Master（主数据库）将用户对数据库更新的操作以二进制格式保存到BinaryLog日志文件中。

Slave（从数据库）上面的I0进程连接上Master， 并请求从指定日志文件的指定位置(或者从最开始的日志)之后的日志内容。

Master接收到来自Slave的I0进程的请求后，通过负责复制的I0进程根据请求信息读取制定日志指定位置之后的日志信息，返回给Slave 的I0进程。返回信息中除了日志所包含的信息之外，还包括本次返回的信息已经到Master端的bin-log文件的名称以及bin-log的位置。

Slave的I0进程接收到信息后，将接收到的日志内容依次添加到Slave端的relay-log文件的最末端，并将读取到的Master端的bin-log的文件名和位置记录到master-info文件中，以便在下一次读取的时候能够清楚的告诉Master “我需要从某个bin- log的哪个位置开始往后的日志内容，请发给我”。

Slave的Sql进程检测到relay-log中新增加了内容后，会马上解析relay- log的内容成为在Master端真实执行时候的那些可执行的内容，并在自身执行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201102143527123.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxNzg2Mjg1,size_16,color_FFFFFF,t_70#pic_center)





##### 3、主从读写分离

只在主服务器上写，在从服务器上读；
主数据库处理事务性查询，从数据库处理SELECT查询；
进行读操作时，是在两个从服务器上轮流读取，利用虚拟模块MySQL-Proxy做读取从服务器时的轮询。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201102143442562.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxNzg2Mjg1,size_16,color_FFFFFF,t_70#pic_center)







### 3、分库分表

###### 

MySQL单库数据量在5000万以内性能比较好,超过阈值后性能会随着数据量的增大而变弱。MySQL单表数据量是500w-1000w之间性能比较好,超过1000w性能也会下降。

因为单个服务的磁盘空间是有限制的,如果并发压力下,所有的请求都访问同一个节点,肯定会对磁盘IO造成非常大的影响。

数据库连接是非常稀少的资源,如果一个库里既有用户、商品、订单相关的数据,当海量用户同时操作时,数据库连接就很可能成为瓶颈。

#### 1、垂直拆分 or 水平拆分？

关系型数据库本身比较容易成为系统瓶颈，单机存储容量、连接数、处理能力都有限。当单表的数据量达到1000W或100G以后，由于查询维度较多，即使添加从库、优化索引，做很多操作时性能仍下降严重。此时就要考虑对其进行切分了，切分的目的就在于减少数据库的负担，缩短查询时间。

数据库分布式核心内容无非就是数据切分（Sharding），以及切分后对数据的定位、整合。数据切分就是将数据分散存储到多个数据库中，使得单一数据库中的数据量变小，通过扩充主机的数量缓解单一数据库的性能问题，从而达到提升数据库操作性能的目的。

数据切分根据其切分类型，可以分为两种方式：垂直（纵向）切分和水平（横向）切分

当我们单个库太大时,我们先要看一下是因为表太多还是数据量太大，如果是表太多,则应该将部分表进行迁移(可以按业务区分),这就是所谓的垂直切分。如果是数据量太大,则需要将表拆成更多的小表,来减少单表的数据量,这就是所谓的水平拆分。


#### **2、垂直**分库****

垂直分库针对的是一个系统中的不同业务进行拆分,比如用户一个库,商品一个库,订单一个库。 一个购物网站对外提供服务时,会同时对用户、商品、订单表进行操作。没拆分之前, 全部都是落到单一的库上的,这会让数据库的单库处理能力成为瓶颈。如果垂直分库后还是将用户、商品、订单放到同一个服务器上,只是分到了不同的库,这样虽然会减少单库的压力,但是随着用户量增大,这会让整个数据库的处理能力成为瓶颈,还有单个服务器的磁盘空间、内存也会受非常大的影响。 所以我们要将其拆分到多个服务器上，这样上面的问题都解决了，以后也不会面对单机资源问题。这种做法与"微服务治理"的做法相似，每个微服务使用单独的一个数据库。

![img](https://img-blog.csdnimg.cn/20200225163309370.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5OTczMDcxMjYz,size_16,color_FFFFFF,t_70)

##### 垂直分表

也就是“大表拆小表”，基于列字段进行的。一般是表中的字段较多，将不常用的， 数据较大，长度较长（比如text类型字段）的字段数据拆分到“扩展表“。一般是针对那种几百列的大表，也避免查询时，数据量太大造成的“跨页”问题。MySQL底层是通过数据页存储的，一条记录占用空间过大会导致跨页（页溢出），造成额外的性能开销（IO操作变多）。另外数据库以页为单位将数据加载到内存中，而页中存储的是行数据，页大小固定，一行数据占用空间越小，页中存储的行数据就越多。这样表中字段长度较短且访问频率较高，内存能加载更多的数据，内存命中率更高，减少了磁盘IO，从而提升了数据库性能。

![img](https://img-blog.csdnimg.cn/20200225163309263.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5OTczMDcxMjYz,size_16,color_FFFFFF,t_70)

##### **垂直拆分的优缺点**

优点：

解决业务系统层面的耦合，业务清晰
与微服务的治理类似，也能对不同业务的数据进行分级管理、维护、监控、扩展等
高并发场景下，垂直切分一定程度的提升IO、数据库连接数、单机硬件资源的瓶颈
缺点：

部分表无法join，只能通过接口聚合方式解决，提升了开发的复杂度
单机的ACID被打破，需要引入分布式事务，而分布式事务处理复杂
依然存在单表数据量过大的问题（需要水平切分）
靠外键去进行约束的场景会受到影响



#### **3、水平拆分**

当一个应用难以再细粒度的垂直切分，或切分后数据量行数巨大，存在单库读写、存储性能瓶颈，这时候就需要进行水平切分了。

水平切分分为库内分表和分库分表，是根据表内数据内在的逻辑关系，将同一个表按不同的条件分散到多个数据库或多个表中，每个表中只包含一部分数据，从而使得单个表的数据量变小，达到分布式的效果。如图所示：

![img](https://img-blog.csdnimg.cn/20200225163309431.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5OTczMDcxMjYz,size_16,color_FFFFFF,t_70)



##### 水平分表

和垂直分表有一点类似,不过垂直分表是基于列的,而水平分表是基于全表的。水平拆分可以大大减少单表数据量,提升查询效率。这里的水平分表指的是在一个数据库进行的库内分表。

库内分表只解决了单一表数据量过大的问题，但没有将表分布到不同机器的库上，因此对于减轻MySQL数据库的压力来说，帮助不是很大，大家还是竞争同一个物理机的CPU、内存、网络IO，最好通过分库分表来解决。



##### 水平分库分表

将单张表的数据切分到多个服务器上去，每个服务器具有相同的库与表，只是表中数据集合不同。 水平分库分表能够有效的缓解单机和单库的性能瓶颈和压力，突破IO、连接数、硬件资源等的瓶颈。



#### **几种常用的分库分表的策略**



##### **5.1** **根据数值范围**

按照时间区间或ID区间来切分。例如：按日期将不同月甚至是日的数据分散到不同的库中；将userId为1~9999的记录分到第一个库，10000~20000的分到第二个库，以此类推。某种意义上，某些系统中使用的"冷热数据分离"，将一些使用较少的历史数据迁移到其他库中，业务功能上只提供热点数据的查询，也是类似的实践。

![img](https://img-blog.csdnimg.cn/20200225163309398.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5OTczMDcxMjYz,size_16,color_FFFFFF,t_70)

这样的优点在于：

单表大小可控
天然便于水平扩展，后期如果想对整个分片集群扩容时，只需要添加节点即可，无需对其他分片的数据进行迁移
使用分片字段进行范围查找时，连续分片可快速定位分片进行快速查询，有效避免跨分片查询的问题。
缺点：

热点数据成为性能瓶颈。连续分片可能存在数据热点，例如按时间字段分片，有些分片存储最近时间段内的数据，可能会被频繁的读写，而有些分片存储的历史数据，则很少被查询



##### ****5.2**根据数值取模**

一般采用hash取模mod的切分方式，例如：将 Customer 表根据 cusno 字段切分到4个库中，余数为0的放到第一个库，余数为1的放到第二个库，以此类推。这样同一个用户的数据会分散到同一个库中，如果查询条件带有cusno字段，则可明确定位到相应库去查询。再比如说有用户表user,将其分成3个表user0,user1,user2.路由规则是对3取模,当uid=1时,对应到的是user1,uid=2时,对应的是user2.

优点：

数据分片相对比较均匀，不容易出现热点和并发访问的瓶颈
缺点：

后期分片集群扩容时，需要迁移旧的数据（使用一致性hash算法能较好的避免这个问题），否则会导致历史数据失效。
容易面临跨分片查询的复杂问题。比如上例中，如果频繁用到的查询条件中不带cusno时，将会导致无法定位数据库，从而需要同时向4个库发起查询，再在内存中合并数据，取最小集返回给应用，分库反而成为拖累。

![img](https://img-blog.csdnimg.cn/20200225163309495.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5OTczMDcxMjYz,size_16,color_FFFFFF,t_70)

https://blog.csdn.net/cy973071263/article/details/104500026?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522163854590416780269887167%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fall.%2522%257D&request_id=163854590416780269887167&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~first_rank_ecpm_v1~rank_v31_ecpm-7-104500026.pc_search_result_cache&utm_term=mysql%E7%9A%84%E5%88%86%E5%BA%93%E5%88%86%E8%A1%A8&spm=1018.2226.3001.4187





https://blog.csdn.net/cy973071263/article/details/104512020