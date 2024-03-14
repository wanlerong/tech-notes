# mysql

## 参考
主要：
- https://dev.mysql.com/doc/refman/5.7/en/innodb-architecture.html
- https://dev.mysql.com/doc/dev/mysql-server/latest/
- https://en.wikipedia.org/wiki/B%2B_tree#Structure

次要：
- https://xiaolincoding.com/mysql/


## innodb 架构

![innodb 架构图](https://dev.mysql.com/doc/refman/5.7/en/images/innodb-architecture-5-7.png)


## 事务、隔离级别、锁、mvcc

ACID 
A: atomicity.
事务是原子化的单元，要么都成功 commit，要么都失败 rollback

C: consistency.
一致性是只数据本身是一致的，不会说积分扣了，东西没到账。
主要指 processing to protect data from crashes.
如：InnoDB doublewrite buffer 一个脏页有16k，而文件系统的写入最小io单位可能是4k或者1k，就意味着可能写到一半发生断电之类的，会导致磁盘脏数据。就不一致了。（为了确保整页写入）
The InnoDB double write buffer helps recover from half-written pages. Whenever InnoDB flushes a page from the buffer pool, it is first written to the double write buffer. Only if the buffer is safely flushed to the disk, will InnoDB write the pages to the disk. So that when InnoDB detects the corruption from the mismatch of the checksum, it can recover from double write buffer.

![InnoDB doublewrite buffer](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240119-192418.png)


InnoDB crash recovery
redo log是在崩溃恢复期间使用。
把那些之前仅仅写到缓冲池中，但还没来得及刷新到disk里的脏页，把结果重新应用一遍。
也就是说，在意外关闭之前没有完成更新的数据文件，将在初始化期间以及连接被接受之前自动恢复。

默认情况下，重做日志在物理上表现为一组名为ib_logfile0和ib_logfile1的文件。
[redo log](http://www.notedeep.com/page/222)



undo log
实现事务回滚，保障事务的原子性。事务处理过程中，如果出现了错误或者用户执 行了 ROLLBACK 语句，MySQL 可以利用 undo log 中的历史数据将数据恢复到事务开始之前的状态。
实现 MVCC（多版本并发控制）关键因素之一。MVCC 是通过 ReadView + undo log 实现的。undo log 为每条记录保存多份历史数据，MySQL 在执行快照读（普通 select 语句）的时候，会根据事务的 Read View 里的信息，顺着 undo log 的版本链找到满足其可见性的记录。


I:: isolation.
http://www.notedeep.com/page/232

事务之间的隔离级别，当多个事务在同一时间执行时，性能和可靠性、一致性之间的平衡/取舍。
通过不同的锁策略实现不同的隔离级别


READ UNCOMMITTED
不加锁，事务可以读取未提交的数据，也称脏读

READ COMMITTED
即使在同一事务中，每次查询都会读取自己的新快照。

UPDATE语句和DELETE语句，InnoDB只锁定索引记录，而不锁定之前的间隔。允许在锁定记录旁边自由插入新记录。
因为禁用了gap lock，所以可能会出现“幻行”，因为其他事务可以将新行插入到间隙中。
对于UPDATE或DELETE语句，InnoDB仅为更新或删除的行保存锁。在MySQL评估WHERE条件之后，释放对不匹配行的记录锁定。大大降低了死锁的可能性。

REPEATABLE READ
这是InnoDB的默认隔离级别。快照读，一个事务内重复执行同样的查询结果相同。
如果使用 select FOR UPDATE 或 LOCK IN SHARE MODE 时 或者 update / delete 时，For a unique index with a unique search condition, InnoDB locks only the index record found, not the gap before it.
For other search conditions, InnoDB locks the index range scanned, using gap locks or next-key locks to block insertions by other sessions into the gaps covered by the range.

SERIALIZABLE
对 select 自动加共享锁
This level is like REPEATABLE READ, but InnoDB implicitly converts all plain SELECT statements to SELECT ... LOCK IN SHARE MODE if autocommit is disabled.


mvcc：
http://www.notedeep.com/page/215

幻行：http://www.notedeep.com/page/239
两次查询出现的行数不一致，或者插入一个之前查不到的id，却报duplicate
在rr级别下，前者mvcc就可以规避，后者可以通过select for update规避（间隙锁+记录锁）
在rc级别下，无法规避，因为rc级别不会加间隙锁


锁：
http://www.notedeep.com/page/230


间隙锁
是“纯粹抑制性的”，这意味着它们的唯一目的是防止其他事务插入到间隙中
rc级别不会加间隙锁

next-key锁
间隙锁和行锁的结合，可以防止幻行
next-key locks是一个索引记录锁，并在索引记录之前的间隙上加上一个gap锁。

两个事务的间隙锁之间是相互兼容的，不会产生冲突。


意向锁：
意向锁是表级锁. 为了支持多粒度锁定，允许行锁和表锁共存。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231227-231056.png)
图中的锁兼容性是针对表级锁的。（例如，一个表级 IX 锁与另一个表级 IX 锁兼容）

要获取行级 X 锁，需要先获取表级的意向锁（如表 IX 锁）

当请求行级X/S锁（例如select ... for update）时，它实际上做的是
---> IX(Table); X(record)
以相反的顺序释放。

如果表级 IX 锁仍未释放，因此其他事务无法在 S 或 X 模式下锁定整个表。
可以让表锁更加有效，有事务将要锁定某些行，此时别的事务不能锁定整个表！
（Mysql不必遍历entrie树来查看是否存在冲突锁）


插入意向锁 Insert Intention Locks
insert intention lock是插入行之前设置的一种gap lock。这个锁发信号通知插入意图，以通知插入到相同gap中的多个事务，如果没有插入gap中相同位置时不需要等待对方。	
可以防止并发插入同一个id的情况。

插入意向锁 会被间隙锁阻塞

自增锁 AUTO-INC Locks
AUTO-INC 锁是一个特殊的表级锁。如果一个事务正在向表中插入值，则任何其他插入的事务都必须等待他插入到该表中。
这样才可以得到连续的主键值。


D: durability
数据的持久化


死锁

死锁产生的四个必要条件：
互斥：一个资源每次只能被一个进程使用(资源独立)。
请求与保持：一个进程因请求资源而阻塞时，对已获得的资源保持不放(不释放锁)。
不剥夺：进程已获得的资源，在未使用之前，不能强行剥夺(抢夺资源)。
循环等待：若干进程之间形成一种头尾相接的循环等待的资源关闭(死循环)。

https://www.xiaolincoding.com/mysql/lock/deadlock.html
https://www.xiaolincoding.com/mysql/lock/show_lock.html

设置事务等待锁的超时时间。当一个事务的等待时间超过该值后，就对这个事务进行回滚，于是锁就释放了，另一个事务就可以继续执行了。在 InnoDB 中，参数 innodb_lock_wait_timeout 是用来设置超时时间的，默认值时 50 秒。

当发生超时后，就出现下面这个提示：

开启主动死锁检测。主动死锁检测在发现死锁后，主动回滚死锁链条中的某一个事务，让其他事务得以继续执行。
将参数 innodb_deadlock_detect 设置为 on，表示开启这个逻辑，默认就开启。

当检测到死锁后，就会出现下面这个提示：

上面这个两种策略是「当有死锁发生时」的避免方式

查看加了什么锁：
select * from performance_schema.data_locks\G;

通过 LOCK_MODE 可以确认是 next-key 锁，还是间隙锁，还是记录锁：
如果 LOCK_MODE 为 X，说明是 next-key 锁；
如果 LOCK_MODE 为 X, REC_NOT_GAP，说明是记录锁；
如果 LOCK_MODE 为 X, GAP，说明是间隙锁；


## 表结构设计
尽可能的小，比如枚举用 tinyint
时间用 unsigned int，只需 4 个字节 可以支持到 2106-02-07 14:28:15，datetime 需要 8 字节，timestamp 4 字节，但只支持到 2038 年
尽可地把字段定义为NOT NULL



## 索引设计，查询优化

http://www.notedeep.com/page/344


### 自适应哈希索引

InnoDB存储引擎会监控对表上各索引页的查询。并建立合适的哈希索引，加速数据页的访问。

特点
哈希索引，查询消耗 O(1)
降低对二级索引树的频繁访问资源。
自适应

缺点
hash自适应索引会占用innodb buffer pool；
自适应hash索引只适合搜索等值的查询，如select * from table where index_col='xxx'，而对于其他查找类型，如范围查找，是不能使用的；


### b+ 树 

二叉查找树：左子树小于根节点，右子树大于根节点。中序遍历后，会得到升序的序列。左右子树的高度可能会相差很大，导致有时查询深度大，性能差。

平衡二叉树：任何节点的子树高度差最大为1，但是因为只有两个子节点，当数据量大时，高度会偏高，"瘦高型"。树越高，每次查询就需要越多次的磁盘IO

B树：矮胖型，Every node has at most m children.
Every internal node has at least ⌈m/2⌉ children.
The root node has at least two children unless it is a leaf.
All leaves appear on the same level.
A non-leaf node with k children contains k−1 keys.

B+树：

![B+ 树](https://upload.wikimedia.org/wikipedia/commons/thumb/3/37/Bplustree.png/800px-Bplustree.png)
和B树的区别：
- 只在叶子节点存储数据，非叶子节点仅存储索引值和指针。内部节点可以存储更多的索引，读取数据时，意味着我们需要访问更少的节点即可
- 索引的key值可以重复（冗余）
- 叶子节点通过指针连接在一起，形成链表
- 从B+树中删除数据更容易并且耗时更少，因为我们只需要从叶节点中删除数据

mysql 中的 B+ 树
树越矮胖，查询时需要的IO次数就会越少。

是B+树的一种变体：
- 叶子节点是双向链表, 比较容易进行范围查询
- 子节点数量等于key的数量，而不是key的数量减一

每个节点存储的都是页，页的默认大小为 16 KB。因为非叶子节点不存实际的数据，假设有一个表的主键是bigint即 8byte，指针大小为6字节，两者加起来总共是14字节。假设一条数据大小为1kb，那一页可以存 16条数据
三层的B+ 树

一、先计算一页的字节大小：16kb * 1024 = 16384 字节。
二、一个索引页节点最多有多少的子节点，16384 字节 / 14 字节 = 1170 

两层 1170 * 16 = 18720 rows
三层 1170 * 1170 * 16 = 21902400 rows

因为索引页不需要存数据，一页可以存更多的索引值, 所以 B+ 树可以比 B树 "更矮胖"。


## 物理层级

系统表空间在 ibdata1 中，里面有 双写缓冲，rollback seg，UNDO SPACE

如果开启了 innodb_file_per_table,则每一个表都会有 ibd file

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240119-213008.png)

tablespace 表空间，包含了多个 segment
segment 包含了一个或多个 extents
extent 为 1MB，有 64 个 page
page 是 16 kb



在表中数据量大的时候，为某个索引分配空间的时候就不再按照页为单位分配了，而是按照区（extent）为单位分配。每个区的大小为 1MB，对于 16KB 的页来说，连续的 64 个页会被划为一个区，这样就使得链表中相邻的页的物理位置也相邻，就能使用顺序 I/O 了。


## change buffer
作用于二级索引（非唯一的），当修改的record不在buffer pool中时，会先写到change buffer中，等之后有读操作将对应page读取到buffer pool之后，再合并过去。
对二级索引的操作通常是随机io，为了避免随机io。

为何唯一索引不能用呢？
比如插入时，pool中不存在，你必须先去disk上加载出来，否则就不知道是否能 insert（是否已经存在）。



## 每个 page 的结构

https://blog.jcole.us/2013/01/07/the-physical-structure-of-innodb-index-pages/

特指 FIL_PAGE_INDEX 类型的 page 

（一个索引就是一个B树（聚簇或二级），一个节点是一个page）

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240119-214402.png)

1、file_header/file_trailer: 
里面存了 page_type, page_type 包含常见的：
FIL_PAGE_INDEX 即： B-tree node，B+树（leaf，non-leaf page）
FIL_PAGE_SDI 即：Tablespace SDI Index page 表结构元数据信息
FIL_PAGE_UNDO_LOG 即：Undo log page 
还存了，上一页和下一页的指针，会指向同一层级的上一页和下一页

2、INDEX header: 
包含了很多索引信息
Index ID: 该page所属的Index的ID
Format Flag: 在这个page 里的 records的格式，有COMPACT和REDUNDANT
Page Direction: 该page正在经历的插入的方向，LEFT（不断插入更小的值），Right（不断插入更大的值，自增），NO_DIRECTION（随机的插入
Number of Inserts in Page Direction: 在这个方向上插入的records数量
level，在 b树中的层级

3、FSEG header:
只有 b+ 树的根节点的 page 才是有效的信息，包含了指向 这个 INDEX 所属文件 segment 的指针。
每个 INDEX 有一个 segment 为叶子节点页面的，另一个 segment 为非叶子节点页面的。

4 System records: infimum and supremum. 
InnoDB has two system records in each page called infimum and supremum. 
该 page 中的最小和最大 record，可以通过 bytes offset 直接访问到。

5 User records:
实际的数据。

6、The page directory:
是该 page 上的 records 的目录，包含了很多个指针，指向该 page 上的 records（每 4 到 8 条记录就会有一个 slot）

## 每行的结构
https://xiaolincoding.com/mysql/base/row_format.html
https://blog.jcole.us/2013/01/10/the-physical-structure-of-records-in-innodb/


Redundant 不是一种紧凑的行格式，基本不用了

默认都是 Compact 类型。

![图](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20240119-232637.png)


变长字段长度列表：
varchar这些，
该record中的那些变长的字段，它们的数据占用的大小存起来，存到「变长字段长度列表」里面

NULL 值列表：
bitmap，一个 bit 表示，一个 field 是否为 null


当一条记录有 9 个字段都是可为 NULL，那么就会创建 2 字节空间的「NULL 值列表」


变长字段长度列表和NULL 值列表的顺序，和field顺序都是相反的。
主要是因为「record header」中指向下一个记录的指针，指向的是下一条记录的「record header」和「真实数据」之间的位置，这样的好处是向左读就是记录头信息，向右读就是真实数据，比较方便。
可以使得位置靠前的记录的真实数据和数据对应的字段长度信息可以同时在一个 CPU Cache Line 中，这样就可以提高 CPU Cache 的命中率。


record header：
delete_mask ：标识此条数据是否被删除。从这里可以知道，我们执行 detele 删除记录的时候，并不会真正的删除记录，只是将这个记录的 delete_mask 标记为 1。
next_record：下一条记录的位置。从这里可以知道，记录与记录之间是通过链表组织的。
在前面我也提到了，指向的是下一条记录的「记录头信息」和「真实数据」之间的位置，这样的好处是向左读就是记录头信息，向右读就是真实数据，比较方便。
record_type：表示当前记录的类型，0表示普通记录，1表示B+树非叶子节点记录，2表示最小记录，3表示最大记录


记录中的真实数据部分。
row_id
如果我们建表的时候指定了主键或者唯一约束列，那么就没有 row_id 隐藏字段了。如果既没有指定主键，又没有唯一约束，那么 InnoDB 就会为记录添加 row_id 隐藏字段。row_id不是必需的，占用 6 个字节。

trx_id
事务id，表示这个数据是由哪个事务生成的。 trx_id是必需的，占用 6 个字节。

roll_pointer
这条记录上一个版本的指针。roll_pointer 是必需的，占用 7 个字节。


每个列的值


## varchar(n) 中 n 最大取值为多少？

所有的列占用的字节长度加起来不能超过 65535 个字节。

在算 varchar(n) 中 n 最大值时，需要用 65535 减去 「变长字段长度列表」和 「NULL 值列表」所占用的字节数的。

假设等于 65532，再考虑字符串编码，在 UTF-8 字符集下，一个字符最多需要三个字节，varchar(n) 的 n 最大取值就是 65532/3 = 21844。


## 如果一个 page 存不了一条记录，即单条记录大于 16 KB

InnoDB 存储引擎会自动将溢出的数据存放到「溢出页」中。

Compact 行格式针对行溢出的处理是这样的：当发生行溢出时，在记录的真实数据处只会保存该列的一部分数据，而把剩余的数据放在「溢出页」中。
然后真实数据处用 20 字节存储指向溢出页的地址，从而可以找到剩余数据所在的页。


## 自增主键的好处

如果我们使用自增主键，那么每次插入的新数据就会按顺序添加到当前索引节点的位置，不需要移动已有的数据，当页面写满，就会自动开辟一个新页面。因为每次插入一条新记录，都是追加操作，不需要重新移动数据，因此这种插入数据的方法效率非常高。

如果我们使用非自增主键，由于每次插入主键的索引值都是随机的，因此每次插入新的数据时，就可能会插入到现有数据页中间的某个位置，这将不得不移动其它数据来满足新数据的插入，甚至需要从一个页面复制数据到另外一个页面，我们通常将这种情况称为页分裂。



## explain 执行计划

possible_keys 字段表示可能用到的索引；
key 字段表示实际用的索引，如果这一项为 NULL，说明没有使用索引；
key_len 表示索引的长度；
rows 表示扫描的数据行数。（只是统计值）
type 表示数据扫描类型，我们需要重点看这个。
type 字段就是描述了找到所需数据时使用的扫描方式是什么，常见扫描类型的执行效率从低到高的顺序为：

All（全表扫描）；坏
index（全索引扫描）；坏
range（索引范围扫描）；用了 where 子句中使用 < 、>、in、between 等，还行
ref（非唯一索引扫描）；
eq_ref（唯一索引扫描）；
const（结果只有一条的主键或唯一索引扫描）。


extra信息：
Using filesort ：当查询语句中包含 group by 操作，而且无法利用索引完成排序操作的时候， 这时不得不选择相应的排序算法进行，甚至可能会通过文件排序，效率是很低的，所以要避免这种问题的出现。
Using temporary：使了用临时表保存中间结果，MySQL 在对查询结果排序时使用临时表，常见于排序 order by 和分组查询 group by。效率低，要避免这种问题的出现。
Using index：所需数据只需在索引即可全部获得，不须要再到表中取数据，也就是使用了覆盖索引，避免了回表操作，效率不错。

## 参数调优

连接参数优化：
对于mysql服务器最大连接数值的设置范围比较理想的是：
Max_used_connections / max_connections  在10%以上
如果在10%以下，说明mysql服务器的max_connections设置过高

max_connections 受限于 mysql 能打开的文件描述符数量 open-files-limit
ini/cnf 参数: open-files-limit。

设置的过高也会有问题，如果在 200 个连接的时候，mysql的iops / cpu 已经打满了。那即使加到1000，也只会有更多的阻塞的链接被建立，会徒增内存和cpu的压力，每个链接都需要资源，会雪上加霜

## 分区表
方式：范围分区，哈希分区，列表分区

优点：
Partitioning makes it possible to store more data in one table than can be held on a single disk or file system partition.

Data that loses its usefulness can often be easily removed from a partitioned table by dropping the partition (or partitions) containing only that data. 

Some queries can be greatly optimized in virtue of the fact that data satisfying a given WHERE clause can be stored only on one or more partitions, which automatically excludes any remaining partitions from the search.

缺点

用于分区的字段，需要包含在每一个唯一索引里，包括主键

分区表设计时，务必明白你的查询条件都带有分区字段，否则会扫描所有分区的数据或索引


在什么场景中会适合用分区呢？

- 业务简单，单表查询，且都会带上分区字段
- 数据需要定期清理，无需保留全部分区


## 分库分表，垂直拆分，水平拆分，读写分离

库的垂直切分，就是划分应用的不同模块。不同模块之间解耦，从业务上看，一个服务的查询只查一个模块。这样不同的模块可以存储在独立的mysql实例中。

表的垂直切分，就是把访问不频繁，字段不耦合的那些字段，根据一些原则拆分到多个表里去。

表的水平切分，通过取模，hash，或者范围划分，将数据路由到相应的子表里。


分库之后可能会引入分布式事务的问题
可以让各个数据库解决自身的事务，然后程序来控制多个数据库上的事务。

跨节点join，排序，分页的问题

## 高可用，分布式扩展


## 复制、备份与恢复

复制使数据能够从一台 MySQL 数据库服务器（称为源）复制到一台或多台 MySQL 数据库服务器（称为副本）。复制默认是异步的；

mysql 8.0 支持不同的复制方法：
1. 传统方法基于从源二进制日志复制事件
2. 基于全局事务标识符(GTID) 

MySQL 8.0除了内置的异步复制之外，还支持半同步复制。
通过半同步复制，在返回到执行事务的会话之前对源块执行提交，直到至少一个副本确认它已接收并记录事务的事件。

MySQL 8.0 还支持延迟复制，即副本故意落后于源至少指定的时间；

复制格式有两种核心类型：基于语句的复制 (SBR)，它复制整个 SQL 语句；以及基于行的复制 (RBR)，它仅复制更改的行。您还可以使用第三种，即基于混合的复制 (MBR)。

连接到源的每个副本都请求二进制日志的副本。也就是说，它从源中提取数据，而不是源将数据推送到副本。

### 复制格式
- 使用基于语句的二进制日志记录时，源将 SQL 语句写入二进制日志。通过在副本上执行 SQL 语句将源复制到副本。这称为 基于语句的复制（可以缩写为 SBR），对应于 MySQL 基于语句的二进制日志记录格式。
- 使用基于行的日志记录时，源会将 事件写入二进制日志，以指示各个表行的更改情况。将源复制到副本的工作方式是将表示表行更改的事件复制到副本。这称为基于行的复制（可以缩写为 RBR）。 基于行的日志记录是默认方法。
- 您还可以将 MySQL 配置为混合使用基于语句和基于行的日志记录，具体取决于哪一种最适合记录更改。这称为 混合格式日志记录。使用混合格式日志记录时，默认使用基于语句的日志。根据某些语句以及所使用的存储引擎，日志在特定情况下会自动切换为基于行。

binlog_format：MIXED,STATEMENT,ROW

基于语句的复制

优点:写入日志文件的数据较少。日志文件包含进行任何更改的所有语句
缺点：依赖于不确定的可加载函数或存储程序的语句，
DELETE and UPDATE statements that use a LIMIT clause without an ORDER BY are nondeterministic（不确定性的）. 
Locking read statements

LOAD_FILE()
UUID(), UUID_SHORT()
USER()
SYSDATE() (unless both the source and the replica are started with the --sysdate-is-now option)
RAND()
VERSION()


row-based 复制

所有更改都可以复制。这是最安全的复制形式。




## 乐观锁 悲观锁

http://www.notedeep.com/page/1231

乐观锁
乐观锁（Optimistic Locking）认为对同一数据的并发操作不会总发生，属于小概率事件，不用每次都对数据上锁，也就是不采用数据库自身的锁机制，而是通过程序来实现。在程序上，我们可以采用版本号机制或者时间戳机制实现。

乐观锁的版本号机制
在表中设计一个版本字段 version，第一次读的时候，会获取 version 字段的取值。然后对数据进行更新或删除操作时，会执行UPDATE ... SET version=version+1 WHERE version=version。此时如果已经有事务对这条数据进行了更改，修改就不会成功。
乐观锁的时间戳机制
时间戳和版本号机制一样，也是在更新提交的时候，将当前数据的时间戳和更新之前取得的时间戳进行比较，如果两者一致则更新成功，否则就是版本冲突。


悲观锁
悲观锁（Pessimistic Locking）也是一种思想，对数据被其他事务的修改持保守态度，会通过数据库自身的锁机制来实现，从而保证数据操作的排它性。

避免等待时间过长，可以把等待锁的超时时间改短到1秒 innodb_lock_wait_timeout=1（默认是50秒）
innodb_thread_concurrency    在InnoDB存储引擎层做“限流”

适用场景

乐观锁适合读操作多的场景，相对来说写的操作比较少。它的优点在于程序实现，不存在死锁问题，不过适用场景也会相对乐观，因为它阻止不了除了程序以外的数据库操作。
悲观锁适合写操作多的场景，因为写的操作具有排它性。


