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
![InnoDB doublewrite buffer](https://images2017.cnblogs.com/blog/1113510/201707/1113510-20170726195345906-321682602.png)

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

![兼容性](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231227-231056.png)
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


