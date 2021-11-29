mysql索引
1、什么是索引

2、索引的数据结构（Hash，B树，B+树）

3、索引的类型

4、索引的设计原则

5、创建索引删除索引

6、最左匹配原则

7、聚簇索引和非聚簇索引

8、回表

9、联合索引



数据结构，为了去快速查找数据

数据结构。     Hash、B树、B+树

### 1、什么是数据库索引

mysql索引就是一种对于表中的一个或者多个字段的值进行排序的一种数据结构，索引可以大大提高MySQL的检索速度。





### 2、索引的优缺点

优点：

索引大大减小了服务器需要扫描的数据量，从而大大加快数据的检索速度，这也是创建索引的最主要的原因。

索引可以帮助服务器避免排序和创建临时表

索引可以将随机IO变成顺序IO

索引对于InnoDB（对索引支持行级锁）非常重要，因为它可以让查询锁更少的元组，提高了表访问并发性

关于InnoDB、索引和锁：InnoDB在二级索引上使用共享锁（读锁），但访问主键索引需要排他锁（写锁）

通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性。

可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。

在使用分组和排序子句进行数据检索时，同样可以显著减少查询中分组和排序的时间。

通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能。

缺点：

创建索引和维护索引要耗费时间，这种时间随着数据量的增加而增加

索引需要占物理空间，除了数据表占用数据空间之外，每一个索引还要占用一定的物理空间，如果需要建立聚簇索引，那么需要占用的空间会更大

对表中的数据进行增、删、改的时候，索引也要动态的维护，这就降低了整数的维护速度

如果某个数据列包含许多重复的内容，为它建立索引就没有太大的实际效果。

对于非常小的表，大部分情况下简单的全表扫描更高效；



索引的性能好主要是看从磁盘读入到内存的IO次数少       就是为什么要选择什么数据结构



### 3、索引的结构：

查找数据，首先是想到顺序查找，但是查找算法时间复杂度是O（n），二分查找树，深度问题。红黑树

1、



红黑树从根到叶子的最长路径不会超过最短路径的2倍。

当插入或删除节点的时候，红黑树的规则可能被打破。这时候就需要做出一些调整， 来继续维持我们的规则。

那么在大规模数据存储的时候，红黑树往往出现由于树的深度过大而造成磁盘IO读写过于频繁，进而导致效率低下的情况。磁盘查找存取的次数往往由树的高度所决定，所以，只要我们通过某种较好的树结构尽量减少树的高度，就可以提高查询性能。B树可以有多个子女，从几十到上千，可以降低树的高度。

归结起来，使用B+树而不是用其他结构的原因就是为了减少磁盘IO的次数，减少数的高度，而B+树就很好的做到了这一点。



B-树（B树）：

一页数据 16K

![img](https://img-blog.csdnimg.cn/2021013023233065.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

- 关键字集合分布在整颗树中；
- 任何一个关键字出现且只出现在一个结点中；
- 搜索有可能在非叶子结点结束；
- 其搜索性能等价于在关键字全集内做一次二分查找；
- 自动层次控制；



B+树；

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210130232533425.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

所有关键字都出现在叶子结点的链表中（稠密索引），且链表中的关键字恰好是有序的；
不可能在非叶子结点命中；
非叶子结点相当于是叶子结点的索引（稀疏索引），叶子结点相当于是存储（关键字）数据的数据层；
每一个叶子节点都包含指向下一个叶子节点的指针，从而方便叶子节点的范围遍历。
更适合文件索引系统；



**B+ 树的优点在于：**

- **IO次数更少**：由于B+树在内部节点上不包含数据信息，因此在内存页中能够存放更多的key。 数据存放的更加紧密，具有更好的空间局部性。因此访问叶子节点上关联的数据也具有更好的缓存命中率。
- **遍历更加方便**：B+树的叶子结点都是相链的，因此对整棵树的遍历只需要一次线性遍历叶子结点即可。而且由于数据顺序排列并且相连，所以便于区间查找和搜索。而B树则需要进行每一层的递归遍历。相邻的元素可能在内存中不相邻，所以缓存命中性没有B+树好。

但是B树也有优点，其优点在于，由于B树的每一个节点都包含key和value，因此经常访问的元素可能离根节点更近，因此访问也更迅速。







HASH：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2021013023265936.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

Hash索引仅仅能满足"=",“IN"和”<=>"查询，不能使用范围查询。也不支持任何范围查询，例如WHERE price > 100。
　　
由于Hash索引比较的是进行Hash运算之后的Hash值，所以它只能用于等值的过滤，不能用于基于范围的过滤，因为经过相应的Hash算法处理之后的Hash值的大小关系，并不能保证和Hash运算前完全一样。



### 索引在磁盘的存储

索引是占据物理空间的，在不同的存储引擎中，索引存在的文件也不同。



存储引擎为MyISAM：    alter table tb_users ENGINE = MYISAM

*.frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
*.MYD：MyISAM DATA，用于存储MyISAM表的数据
*.MYI：MyISAM INDEX，用于存储MyISAM表的索引相关信息



存储引擎为InnoDB：

*.frm：与表相关的元数据信息都存放在frm文件，包括表结构的定义信息等
*.ibd：InnoDB DATA，表数据和索引的文件。该表的索引(B+树)的每个非叶子节点存储索引，叶子节点存储索引和索引对应的数据





### 索引的分类：

功能分类：

主键索引；

唯一索引：

普通索引：

前缀索引：

全文索引：



列数分类：

单列索引：

联合索引：

最左匹配原则：



### 物理分类加Innodb索引的实现：

**聚簇索引：**

聚簇索引（clustered index）不是单独的一种索引类型，而是一种数据存储方式。这种存储方式是依靠B+树来实现的，根据表的主键构造一棵B+树且B+树叶子节点**存放的都是表的行记录数据时，方可称该主键索引为聚簇索引**。聚簇索引也可理解为将数据存储与索引放到了一块，找到索引也就找到了数据。**每张表最多只能拥有一个聚簇索引。**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201102538557.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

因为InnoDB的数据文件（.idb）按主键聚集，所以InnoDB必须有主键（MyISAM可以没有），如果没有显示指定主键，则选取首个为唯一且非空的列作为主键索引，如果还没具备，则MySQL自动为InnoDB表生成一个隐含字段作为主键，这个字段长度为6个字节，类型为长整形。



非聚簇索引：

非聚簇索引：**数据和索引是分开的，B+树叶子节点存放的不是数据表的行记录**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201102648935.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

在聚簇索引之外创建的索引（不是根据主键创建的）称之为辅助索引，辅助索引访问数据总是需要二次查找。辅助索引叶子节点存储的不再是行数据记录，而是主键值。首先通过辅助索引找到主键值，然后到主键索引树中通过主键值找到数据行。



MYISAM

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201170401133.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)

可以看到叶子节点的存放的是数据记录的地址。也就是说索引和行数据记录是没有保存在一起的，所以MyISAM的主键索引是非聚簇索引。

只是主索引要求key是唯一的，而辅助索引的key可以重复。 MyISAM辅助索引也是非聚簇索引。

MyISM使用的是非聚簇索引，表数据存储在独立的地方，这两棵（主键和辅助键）B+树的叶子节点都使用一个地址指向真正的表数据。由于索引树是独立的，通过辅助键检索无需访问主键的索引树。

对比图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210201172450840.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)





### explain命令的参数

https://www.cnblogs.com/ambitionutil/p/11278600.html

expain出来的信息有10列，分别是id、select_type、table、type、possible_keys、key、key_len、ref、rows、Extra,

id：**SQL执行的顺序的标识,SQL从大到小的执行**

select_type:***\*查询中每个select子句的类型\****

**table**:数据在那个表

**type**：查询语句的访问类型，其实就是性能，也是我们比较关注的   **ALL, index, range, ref, eq_ref, const, system, NULL（从左到右，性能从差到好）**

ALL：Full Table Scan， MySQL将遍历全表以找到匹配的行

index: Full Index Scan，index与ALL区别为index类型只遍历索引树

range:只检索给定范围的行，使用一个索引来选择行

ref: 

eq_ref: 类似ref，区别就在使用的索引是唯一索引，对于每个索引键值，表中只有一条记录匹配，简单来说，就是多表连接中使用primary key或者 unique key作为关联条件

const、system: 当MySQL对查询某部分进行优化，并转换为一个常量时，使用这些类型访问。如将主键置于where列表中，MySQL就能将该查询转换为一个常量,system是const类型的特例，当查询的表只有一行的情况下，使用system

NULL: MySQL在优化过程中分解语句，执行时甚至不用访问表或索引，例如从一个索引列里选取最小值可以通过单独索引查找完成。



一般情况下最好的是const  最差为ALL



**possible_keys**：**指出MySQL能使用哪个索引在表中找到记录，查询涉及到的字段上若存在索引，则该索引将被列出，但不一定被查询使用**



key：**key列显示MySQL实际决定使用的键（索引）**



**key_len**：**表示索引中使用的字节数，可通过该列计算查询中使用的索引的长度（key_len显示的值为索引字段的最大可能长度，并非实际使用长度，即key_len是根据表定义计算而得，不是通过表内检索出的）**



**ref**：**上述表的连接匹配条件，即哪些列或常量被用于查找索引列上的值**，常见的值有 const, func, NULL,

​         当 key 列为 NULL ， ref 列也相应为 NULL。

​           这次key 列使用了主键索引，where id=1 中 1 为常量， ref 列的 const 便是指这种常量。



rows：**表示MySQL根据表统计信息及索引选用情况，估算的找到所需的记录所需要读取的行数**



**Extra**：**包含MySQL解决查询的详细信息**



### 最左匹配原则：

在MySQL建立联合索引时会遵守最左前缀匹配原则，即最左优先（查询条件精确匹配索引的左边连续一列或几列，则构建对应列的组合索引树），在检索数据时也从联合索引的最左边开始匹配。



只要筛选条件中含有组合索引最左边的列但不含有主键搜索键的时候，至少会构建包含组合索引最左列的索引树。

select * from tb_index  where  a=1;

select * from tb-index where d=1;



查询列都是组合索引列且筛选条件全是组合索引列时，会构建满列组合索引树（index(a,b,c) ）

select a,b,c from tb_index where b = 1;(覆盖索引)



筛选条件包含普通搜索键但没包含组合索引列最左键，不会构建组合索引树

select  * from tb_index where b = 1

Select * from tb_index where b=1 and c = 1



如果筛选条件全是组合索引最左**连续列**作为搜索键，将构建连续列组合索引树。（比如：index(a,b)却不能index(a,c)）

select * from tb_index where a = 1 and b=1 走

select* from tb_index where a =1 and c= 1 不走



MySQL查询优化器会优化and连接，将组合索引列规则排号。（比如：b and a 等同于 a and b）





### 覆盖索引和回表



覆盖索引其形式就是，搜索的索引键中的字段恰好是查询的字段（或是组合索引键中的其它字段）。覆盖索引的查询效率极高，原因在与其不用做回表查询。             type index





![在这里插入图片描述](https://img-blog.csdnimg.cn/20210206194334864.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdmZWlqaXU=,size_16,color_FFFFFF,t_70)



索引失效：





### 索引调优：（避免不走索引的语句）

1、**尽量避免在字段开头模糊查询，会导致数据库引擎放弃索引进行全表扫描**

SELECT * FROM t WHERE username LIKE '%陈%'

优化方式：尽量在字段后面使用模糊查询。如下：

```vsql
SELECT * FROM t WHERE username LIKE '陈%'
```

2、**尽量避免使用in 和not in，会导致引擎走全表扫描。**如下：

```sql
SELECT * FROM t WHERE id IN (2,3)
```

优化方式：如果是连续数值，可以用between代替。如下：

```sql
SELECT * FROM t WHERE id BETWEEN 2 AND 3
```

3、尽量避免使用 or，会导致数据库引擎放弃索引进行全表扫描。如下：

SELECT * FROM t WHERE id = 1 OR id = 3
优化方式：可以用union代替or。如下：

SELECT * FROM t WHERE id = 1
   UNION
SELECT * FROM t WHERE id = 3

4、**尽量避免进行null值的判断，会导致数据库引擎放弃索引进行全表扫描。**如下：

```sql
SELECT * FROM t WHERE score IS NULL
```

优化方式：可以给字段添加默认值0，对0值进行判断。如下：

```sql
SELECT * FROM t WHERE score = 0
```

5、**尽量避免在where条件中等号的左侧进行表达式、函数操作，会导致数据库引擎放弃索引进行全表扫描。**

可以将表达式、函数操作移动到等号右侧。如下：

-- 全表扫描

SELECT * FROM T WHERE score/10 = 9

-- 走索引

SELECT * FROM T WHERE score = 10*9

6、**查询条件不能用 <> 或者 !=**

7、**where条件仅包含复合索引非前置列**

8、**隐式类型转换造成不使用索引** 

9、**order by 条件要与where中条件一致，否则order by不会利用索引进行排序**

这是属于Mysql优化中的Sql优化的索引优化

















