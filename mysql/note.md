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

自增锁 AUTO-INC Locks
AUTO-INC 锁是一个特殊的表级锁。如果一个事务正在向表中插入值，则任何其他插入的事务都必须等待他插入到该表中。
这样才可以得到连续的主键值。


D: durability
数据的持久化

## 表结构设计
尽可能的小，比如枚举用 tinyint
时间用 unsigned int，只需 4 个字节 可以支持到 2106-02-07 14:28:15，datetime 需要 8 字节，timestamp 4 字节，但只支持到 2038 年
尽可地把字段定义为NOT NULL



## 索引设计，查询优化

http://www.notedeep.com/page/344

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


## 分区表

## 分库分表，垂直拆分，水平拆分，读写分离

## 高可用，分布式扩展

## 复制、备份与恢复


# 其他

## 查询缓存


