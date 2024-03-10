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

### 复制
At the base of Redis replication，there is a leader follower (master-replica) replication that is simple to use and configure. 它允许 Redis 副本实例成为主实例的精确副本。每次链接断开时，副本都会自动重新连接到主服务器，并且无论主服务器发生什么情况，都会尝试成为主服务器的精确副本。

该系统使用三种主要机制来工作：

1. 当主实例和副本实例连接良好时，主实例通过向副本发送命令流来复制主实例中由于以下原因而对数据集发生的影响来保持副本更新：客户端写入、key过期或驱逐、更改主数据集的任何其他操作。
2. 当主服务器和副本服务器之间的链路由于网络问题或在主服务器或副本服务器中检测到超时而断开时，副本服务器会重新连接并尝试继续进行部分重新同步：这意味着它将尝试仅获取该部分在断开连接期间丢失的命令流。
3. 当无法进行部分重新同步时，副本将请求完全重新同步。这将涉及一个更复杂的过程，其中主服务器需要创建其所有数据的快照，将其发送到副本，然后在数据集发生变化时继续发送命令流。

redis 默认使用异步复制，高性能和低延迟的。

However, Redis replicas asynchronously acknowledge the amount of data they received periodically with the master.（redis 副本会定期的异步的向master发送信息，表示自己确认接收到的数据 key 的量）（但如果需要，也可以配置同步复制）

客户端可以使用该命令请求同步复制某些数据WAIT。但WAIT只能确保其他 Redis 实例中存在指定数量的已确认副本，并不能将一组 Redis 实例转变为具有强一致性的 CP 系统：在故障转移期间，已确认写入仍可能会丢失，具体取决于Redis持久化的精确配置。然而，由于WAIT。某些难以触发的故障模式，故障事件后丢失写入的概率大大降低。


副本还可以以级联结构连接到其他副本。从Redis 4.0开始，所有子副本都会从主副本接收完全相同的复制流。


当使用复制时，强烈建议开启持久化。如果不能开启，则要避免redis崩溃后自动重启，因为：
```
We have a setup with node A acting as master, with persistence turned down, and nodes B and C replicating from node A.
Node A crashes, however it has some auto-restart system, that restarts the process. However since persistence is turned off, the node restarts with an empty data set.
Nodes B and C will replicate from node A, which is empty, so they'll effectively destroy their copy of the data.
```

### 复制是如何工作的

Every Redis master has a replication ID
和一个offset，偏移量随着生成的要发送到副本的复制流的每个字节而递增。
这两个字段标识了主数据集的精确版本。

当副本连接到主服务器时，它们使用该PSYNC命令发送其旧的主服务器复制 ID 以及迄今为止处理的偏移量。这样master就可以只发送所需的增量部分。但是，如果主缓冲区中没有足够的积压，或者副本引用不再已知的历史记录（复制 ID），则会发生完全重新同步：在这种情况下，副本将获得数据集的完整副本，从头开始​​。

全量同步的过程：
主设备启动后台保存过程以生成 RDB 文件。同时，它开始缓冲从客户端接收到的所有新写入命令。当后台保存完成后，主服务器将数据库文件传输到副本，副本将其保存在磁盘上，然后将其加载到内存中。然后，主服务器会将所有缓冲的命令发送到副本服务器。这是作为命令流完成的，并且采用与 Redis 协议本身相同的格式。


### 复制ID

复制 ID 基本上标记了数据集的给定历史记录。每次实例作为主实例重新启动，或者副本升级为主实例时，都会为此实例生成一个新的复制 ID。连接到主服务器的副本将在握手后继承其复制 ID。因此，具有相同 ID 的两个实例是相关的，因为它们拥有相同的数据，但可能在不同的时间。对于给定的历史记录（复制 ID），该偏移量充当逻辑时间来了解谁拥有最新的数据集。

Redis 实例之所以有两个复制 ID，是因为副本会提升为主服务器。故障转移后，升级的副本仍需要记住其过去的复制 ID，因为该复制 ID 是前一个主副本的复制 ID。这样，当其他副本与新主服务器同步时，它们将尝试使用旧主服务器复制 ID 执行部分重新同步。这将按预期工作，因为当副本升级为主服务器时，它将其辅助 ID 设置为其主 ID，并记住发生此 ID 切换时的偏移量。稍后它将选择一个新的随机复制 ID，因为新的历史记录开始了。当处理新的副本连接时，主节点会将它们的 ID 和偏移量与当前 ID 和辅助 ID 进行匹配（为了安全起见，最多达到给定的偏移量）。简而言之，这意味着在故障转移后，连接到新升级的主服务器的副本不必执行完全同步。

如果您想知道为什么升级为主服务器的副本需要在故障转移后更改其复制 ID：有可能由于某些网络分区，旧主服务器仍作为主服务器工作：保留相同的复制 ID 会违反以下事实：任何两个随机实例的相同 ID 和相同偏移量意味着它们具有相同的数据集。

Redis 实例之所以有两个复制 ID，是因为副本会提升为主服务器。故障转移后，升级的副本仍需要记住其过去的复制 ID，因为该复制 ID 是前一个主副本的复制 ID。这样，当其他副本与新主服务器同步时，它们将尝试使用旧主服务器复制 ID 执行部分重新同步。这将按预期工作，因为当副本升级为主服务器时，它将其辅助 ID 设置为其主 ID，并记住发生此 ID 切换时的偏移量。稍后它将选择一个新的随机复制 ID，因为新的历史记录开始了。当处理新的副本连接时，主节点会将它们的 ID 和偏移量与当前 ID 和辅助 ID 进行匹配（为了安全起见，最多达到给定的偏移量）。简而言之，这意味着在故障转移后，连接到新升级的主服务器的副本不必执行完全同步。


配置

配置基本的 Redis 复制很简单：只需将以下行添加到副本配置文件中：
replicaof 192.168.1.1 6379


### redis cluster
https://redis.io/docs/management/scaling/

数据自动在多个 Redis 节点之间分片, 有一定程序的可用性。

自动将数据集分割到多个节点之间。
当部分节点遇到故障或无法与集群的其余节点通信时，继续操作。

16379：的集群总线端口，二进制协议的节点到节点的通信通道，在节点之间交换信息。节点使用集群总线进行故障检测、配置更新、故障转移授权等。
### Redis集群数据分片
计算给定键的哈希槽，我们只需对键的 CRC16 取模 16384 即可
Redis 集群中的每个节点负责哈希槽的子集，因此，例如，您可能有一个包含 3 个节点的集群，其中：

- 节点 A 包含从 0 到 5500 的哈希槽。
- 节点 B 包含从 5501 到 11000 的哈希槽。
- 节点 C 包含从11001到16383的哈希槽。

这使得添加和删除集群节点变得容易。例如，如果我想添加一个新节点 D，我需要将一些哈希槽从节点 A、B、C 移动到 D。

将哈希槽从一个节点移动到另一个节点不需要停止任何操作

redis 事务以及 lua 脚本，需要涉及的 key 属于同一个hash slot。可以使用花括号这种方式 {uniqueid}

### Redis 的主从模型
Redis 集群使用主副本模型，其中每个哈希槽都有 1 个（主节点本身）到 N 个副本 (N-1额外的副本节点）。

在我们的具有节点 A、B、C 的示例集群中，如果节点 B 发生故障，集群将无法继续，因为我们不再有办法为 5501-11000 范围内的哈希槽提供服务。

但是，当集群创建时（或稍后），我们为每个主节点添加一个副本节点，这样最终的集群由 A、B、C 为主节点，A1、B1、C1 为主节点组成。副本节点。这样，即使节点 B 发生故障，系统也可以继续运行。

节点B1复制B，并且B出现故障，集群会将节点B1提升为新的master并继续正确运行。

但需要注意的是，如果节点B和B1同时出现故障，Redis Cluster将无法继续运行。

### redis cluster 不保证强一致性
可能会丢失数据，原因是异步复制。客户端向主节点B发起写入，B在回复客户端之前，不会等待B1、B2、B3的确认。所以可能在写入B之后，发生崩溃，然后B1提升为master，但是B1可能还没有完成最新的复制。

可以通过WAIT命令实现同步写入，但通常会导致性能过低，即使使用同步复制，Redis Cluster 也不会实现强一致性：在更复杂的故障场景下，始终有可能无法接收写入的副本将被选为主。

还有另一种值得注意的情况，Redis 集群会丢失写入，这种情况发生在网络分区期间，客户端与少数实例（至少包括一个主实例）隔离。

以我们的 6 节点集群为例，由 A、B、C、A1、B1、C1 组成，具有 3 个主节点和 3 个副本节点。还有一个客户端，我们称之为 Z1。

分区发生后，有可能在分区的一侧有A、C、A1、B1、C1，而在另一侧有B和Z1。

Z1 仍然能够写入 B，B 将接受其写入。如果分区在很短的时间内修复，集群将正常运行。但是，如果分区持续了足够的时间，使 B1 晋升为分区多数侧的主控，则 Z1 在此期间发送到 B 的写入将会丢失。

Z1 能够发送到 B 的写入量有一个最大窗口：如果分区的多数端已经过了足够的时间来选举副本作为主节点，则少数端的每个主节点将停止接受写入。

这个时间是 Redis Cluster 的一个非常重要的配置指令，称为节点超时。

节点超时后，主节点将被视为发生故障，并且可以由其副本之一替换。类似地，在节点超时过后，如果主节点无法感知大多数其他主节点，则它会进入错误状态并停止接受写入。

cluster-node-timeout<milliseconds>：Redis 集群节点不可用且不被视为失败的最长时间。如果主节点在指定时间内无法访问，则其副本将对其进行故障转移。该参数控制Redis集群中的其他重要的事情。值得注意的是，在指定时间内无法到达大多数主节点的每个节点将停止接受查询。

cluster-slave-validity-factor<factor>：如果设置为零，副本将始终认为自己有效。例如，如果节点超时设置为 5 秒，有效性因子设置为 10，则与主服务器断开连接超过 50 秒的副本将不会尝试对其主服务器进行故障转移。请注意，如果没有能够对其进行故障转移的副本，则任何不为零的值都可能导致 Redis 集群在主服务器发生故障后不可用。在这种情况下，只有当原始主节点重新加入集群时，集群才会恢复可用。


创建Redis集群的要求
要创建集群，首先需要有一些以集群模式运行的空 Redis 实例。

至少在redis.conf文件中设置以下指令：
```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes


要创建集群，请运行：

redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 \
127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 \
--cluster-replicas 1

这里使用的命令是create，因为我们要创建一个新的集群。该选项--cluster-replicas 1意味着我们希望为每个创建的主节点创建一个副本。
```


## redis 过期删除


## redis 内存淘汰
refcount：引用计数
lru：对象最后一次被访问的时间，长期未被访问的会优先释放

仅共享整数值的字符串，因为容易验证是否一致
![encode](https://raw.githubusercontent.com/wanlerong/tech-notes/master/imgs/WX20231229-174926.png)


## redis缓存设计，缓存击穿，穿透，雪崩；缓存一致性

## 实战