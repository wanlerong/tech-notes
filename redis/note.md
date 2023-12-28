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

### 跳跃表

有序的，每个节点维持多个指向其他节点的指针
查找效率，平均O(logN),最坏O(N)
sorted set 的底层实现（当元素数量较多/或member是比较长的字符串时）


## redis 线程模型



## redis 持久化

## 分布式锁

## redis 集群, 高可用

## redis 过期删除与内存淘汰

## redis缓存设计，缓存击穿，穿透，雪崩；缓存一致性

## 实战