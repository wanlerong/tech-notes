# redis

## redis 数据结构

### string
sds：结构为 len, free, buf（指向字节数组的指针，所以可以保存二进制数据）

场景：cache，counter，Bitwise operations，分布式锁

bitmap：Bitmaps are not an actual data type, but a set of bit-oriented operations defined on the String type which is treated like a bit vector.

表示第123个传感器，在该时间，ping了。一个bit表示一个传感器
SETBIT pings:2024-01-01-00:00 123 1



### list 链表

Redis lists are linked lists of string values. 底层是一个双向链表

用于
- Implement stacks and queues.
- Build queue management for background worker systems.
（发布与订阅，慢查询，监视器等）

![链表](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231228-180433.png)

### hash 

Redis hashes are record types structured as collections of field-value pairs. You can use hashes to represent basic objects and to store groupings of counters, among other things.

![哈希表](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-014227.png)

sizemask 哈希表大小掩码，总是等于size-1，用于计算索引值 (可以控制索引出来的值会在数组范围内，链表的数组)

hash = dict->type->hashFunction(key)
index = hash & sizemask

trehashidx rehash索引，当不在rehash时，值为-1

ht 用于做rehash，是两个元素的数组

负载因子 = used（已有的键值对数量）/ size（链表的数组的大小）

### sorted set

A Redis sorted set is a collection of unique strings (members) ordered by an associated score. When more than one string has the same score, the strings are ordered lexicographically. Some use cases for sorted sets include:

Leaderboards. For example, you can use sorted sets to easily maintain ordered lists of the highest scores in a massive online game.
Rate limiters. In particular, you can use a sorted set to build a sliding-window rate limiter to prevent excessive API requests.

Zset 类型的底层数据结构是由压缩列表或跳表实现的：

如果有序集合的元素个数小于 128 个，并且每个元素的值小于 64 字节时，Redis 会使用压缩列表作为 Zset 类型的底层数据结构；
如果有序集合的元素不满足上面的条件，Redis 会使用跳表作为 Zset 类型的底层数据结构；

同时使用跳跃表和字典实现，因为跳表可以更好地支持范围查询，字典可以支持单个查询

#### 跳跃表

有序的，每个节点维持多个指向其他节点的指针
查找效率，平均O(logN),最坏O(N)
sorted set 的底层实现（当元素数量较多/或member是比较长的字符串时）


![跳表](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-160252.png)

![跳表node](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-160510.png)

查找目标节点的过程中，访问的每个层的跨度的累计，就是目标节点的排位。

每个节点的层高，都是1 ~ 32直接的随机数

### set

A Redis set is an unordered collection of unique strings (members). You can use Redis sets to efficiently:

Track unique items (e.g., track all unique IP addresses accessing a given blog post).
Represent relations (e.g., the set of all users with a given role).
Perform common set operations such as intersection, unions, and differences.

#### intset
当set中只有整数，且元素不多时。
![intset](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-163025.png)
encodexxx_INT16 表示存的是16位的整数，最大为32767

contents 数组按从小到大的顺序存

### 压缩列表 ziplist

https://redis.com/glossary/redis-ziplist/

大于时将不再用 ziplist

list-max-ziplist-entries: 512
list-max-ziplist-value: 64
(Limits for ziplist usage with LISTs)
hash-max-ziplist-entries: 512
hash-max-ziplist-value: 64
(Limits for ziplist usage with HASHes; previous versions of Redis used different names and encodings for this)
zset-max-ziplist-entries: 128
zset-max-ziplist-value: 64
(Limits for ziplist usage with ZSETs)


是list和hash的底层实现之一，当list元素较少，且每个项要么是小整数值，要么是长度短的字符串时，redis会用压缩列表来作为list的实现。hash也是同理

为了节约内存，连续内存块组成的顺序型数据结构

![ziplist](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-165208.png)

Each entry represents an element and contains the following components:
Prevlen: This field stores the length of the previous entry. It allows for efficient traversal in both forward and backward directions within the ziplist.
Entrylen: This field stores the length of the current entry.
Content: This field contains the actual data of the element, such as a string, integer, or floating-point value.

为何能节省空间？
ziplist uses two variable length encoded numbers to replace the two pointers, 如果content长度很小，那两个长度值，只需要2字节。但是两个指针将花费16个字节（在64位系统上）
就能至少节省14字节（还可节省其他元数据）。

但是如果内容本身很大，节省的这些就没有太大意义。
并且压缩链表会给cpu增加负担，每次写都要分配内存。（写少，读多，且cpu压力不大，内存压力大时，可以适当加大一些ziplist阈值的配置）
On the other hand, since ziplist saves all entries in contiguous memory in a compact way, it has to do memory reallocation for almost every write operation. That's very CPU inefficient. Also encoding and decoding those variable length encoded number cost CPU.

### 对象
type：string,list,hash,set,zset
encode: 不同的type可以由不同的encode实现
![encode](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-173602.png)



## 事务
multi/exec 和 lua 脚本的区别
lua脚本可以获得中间结果，当你需要再事务中依赖某个中间结果时，只能用lua
multi/exec 只会在全部结束时，返回结果的数组
都不支持回滚

## redis 线程模型


## redis 持久化

rdb

## aof 重写

子进程进行AOF重写期间，服务器进程（父进程）可以继续处理命令请求。
子进程带有服务器进程的数据副本，使用子进程而不是线程，可以在避免使用锁的情况下，保证数据的安全性。

AOF缓冲区的内容会定期被写入和同步到AOF文件，对现有AOF文件的处理工作会如常进行。从创建子进程开始，服务器执行的所有写命令都会被记录到AOF重写缓冲区里面。当子进程完成AOF重写工作之后，它会向父进程发送一个信号，父进程在接到该信号之后，会调用一个信号处理函数，并执行以下工作：
    1）将AOF重写缓冲区中的所有内容写人到新AOF文件中，这时新AOF文件所保存的数据库状态将和服务器当前的数据库状态一致。
    2）对新的AOF文件进行改名，原子地（atomic）覆盖现有的AOF文件，完成新两个AOF文件的替换。

在执行BGREWRITEAOF命令时，Redis服务器会维护一个AOF重写缓冲区，该缓冲区会在子进程创建新AOF文件期间，记录服务器执行的所有写命令。当子进程完成创建新AOF文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新AOF文件的末尾，使得新旧两个AOF文件所保存的数据库状态一致。最后，服务器用新的AOF文件替换旧的AOF文件，以此来完成AOF文件重写操作。


## 分布式锁

## redis 集群, 高可用

## redis 过期删除


## redis 内存淘汰
refcount：引用计数
lru：对象最后一次被访问的时间，长期未被访问的会优先释放

仅共享整数值的字符串，因为容易验证是否一致
![encode](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-174926.png)


## redis缓存设计，缓存击穿，穿透，雪崩；缓存一致性

## 实战