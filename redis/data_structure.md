# redis数据结构
## 参考
[pdai Redis进阶](https://pdai.tech/md/db/nosql-redis/db-redis-x-redis-object.html)

[Redis 源码剖析与实战](https://time.geekbang.org/column/intro/100084301)

[小林coding redis](https://xiaolincoding.com/redis/data_struct/command.html)

[HyperLogLog算法原理](https://juejin.cn/post/6844903785744056333)

## 底层结构
### redisObject
redisObject为不同类型的数据提供了统一的接口，使Redis可以用一致的方式处理不同类型的数据。
```c
struct redisObject {
    unsigned type:4; // 数据类型
    unsigned encoding:4; // 编码方式
// LRU 记录最末一次访问时间; 或者 LFU（8位频率，16位访问时间）
    unsigned lru:LRU_BITS;
    int refcount; // 引用计数，用于垃圾回收
    void *ptr; // 指向底层数据结构实例
};
```

![整体结构](https://pdai.tech/images/db/redis/db-redis-object-2-2.png)

### 简单动态字符串 SDS
redis自己封装的字符串类型，四个字段（len现有长度，alloc分配的空间长度，flags SDS的类型，buf[]字符数组）

#### 与 C 语言原生字符串相比的优势
alloc和len字段
1. 避免了C语言中获取字符串长度需要**遍历**的缺点
2. 可以保存二进制数据（C语言遇到`\0`就表示字符串结束了）
3. 字符串拼接时可以判断内存空间是否满足需求，避免溢出
4. 字符串缩短时可以**惰性**释放空间，提高性能

flags字段
1. flags类型是针对不同大小的字符串选择了不同比特位的int型变量，节约了内存空间。

PS：以上也是 golang 中*切片*的设计思想（go中string底层没有cap字段，拼接是开辟新内存然后依次copy过去）

编译指令
在分配内存时采用**紧凑(packed)**的方式，避免内存对齐，同样为了节约内存。

### ziplist 压缩列表
以动态数组的**紧凑布局**保存**int或者string**。在7.0被废弃。

整体结构：`<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>`
* zlbytes：列表内存占用大小
* zltail：尾节点内存偏移量
* zllen：节点总数
* zlend：0xFF，末尾标记

entry结构：`<prevlen> <encoding> <entry-data>`
* prevlen：前一个entry大小
* encoding：data存储的元素类型和长度
* entry-data：实际数据

#### 遍历
从左向右：计算prevlen占用内存，进而得到encoding起始地址，根据encoding编码可得数据大小，从而找到下一个entry起始地址。
从右向左：通过prevlen和尾节点地址可以依次推导出上一个节点的地址

#### 优势
相对于普通数组，不需要预分配内存，且每个节点大小不必相同（通过encoding细化存储大小），**节约内存**。

#### 缺点
1. 每次写操作（扩容缩容）都会进行内存分配
2. 因为 encoding 对于不同类型不同大小的数据进行了细化，当数据大小变化时（新增节点导致prevlen内存占用变化），可能会引发**连锁更新**
3. 元素越多对于列表中间节点的查找时间复杂度越高

### listpack 紧凑列表
listpack是redis5.0中针对ziplist**连锁更新**的问题进行改良而来的数据结构，因此同样也属于紧凑布局。

基本结构和ziplist相似
整体结构：`<lpbytes> <lplen> <entry> <entry> ... <entry> <lpend>`
entry结构：`<encoding> <entry-data> <entry-len>`

* 不同：整体结构中移除了尾节点偏移量，entry结构中改为保存**当前**entry中 `encoding+data` 的总长度，并且为了从右向左遍历使用了**大端存储模式**并将entry-len放到entry末尾。

#### 遍历
从左向右：解析encoding可以得到元素类型和长度，并且可以进一步计算得到len本身的长度，从而找到下一个entry起始地址。
从右向左：通过lpbytes总长度直接定位到尾部结束标记，然后从右到左逐个字节读取entry-len（当读到的字节最高位为0表示该结束了）,获得entry前两部分大小，从而可以定位到前一项entry的末尾，如此往复。

### quicklist 快表
redis7.0之前通过双向链表+ziplist实现，7.0之后改为双向链表+listpack。

quicklist每个节点都保存一个ziplist/listpack，在添加元素时会检查插入位置的压缩列表是否能容纳该元素，如果能容纳就直接保存到 quicklistNode 结构里的压缩列表，如果不能容纳，才会新建一个新的 quicklistNode 结构。具体一个列表中可以容纳几个元素可以在配置文件中设置。

#### 优势
1. 综合了链表和数组的优势，链表：可以动态增减节点；数组：内存连续节省空间
2. 避免列表中节点过多导致的查询时间复杂度变高，分散列表负载
3. 可以根据性能动态调整（修改配置文件），时间换空间或者空间换时间

#### 缺点
由于融合了两种数据结构所以维护成本更高，7.0之前使用ziplist结构无法避免连锁更新问题。

### 哈希表
桶数组+拉链法。插入一对k-v数据时，计算key的hash值并找到对应桶索引，遇到冲突则新建一个entry连接到链表尾。

entry是链表中的节点，每个节点保存着key、value（存值或者指针）以及指向下一个节点的指针。

一个hash结构有两个hash表，一个用来存储kv，另一个为空，在扩容时使用。

#### 哈希表的扩容
触发条件：
1. 服务器目前没有执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于1。
2. 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，并且负载因子大于等于5。
ps：`负载因子 = 哈希表已保存节点数量 / 哈希表大小`

**渐进式**rehash过程：
为了避免大量数据集中拷贝带来的性能影响，故使用渐进式rehash，具体过程为：
1. 为那个空的哈希表分配两倍的内存空间
2. 在rehash期间，增删改查操作都会在执行对应操作以外，按索引顺序一个一个将索引位置的数据从旧哈希表迁移到新表
3. 当全部迁移完了，释放旧表的内存空间

* 和golang的map扩容思想非常相似，不过go中一个桶容量有限制，多出的放溢出桶，所以go中还存在等量扩容的情况

### intset 整数集
三个字段：编码方式encoding、元素数量length、保存元素的数组contents[]。整数集实际上就是一个**递增**int数组。

在添加每个元素时，intset会始终保持有序，目的是便于查找。

其中contetns数组的int类型（int16、int32、int64）会根据元素的大小确定。

适用于只存储int值且存储数量较少的情况。

#### 升级
当加入的新元素类型比现有的元素类型大时，需要对数组进行扩容：
1. 计算类型升级后数组所需要的空间，追加分配到原数组末尾。
2. 从右向左依次转换原数组中的元素
3. 添加新元素到数组末尾

* 优势：根据元素大小动态选择数组类型，可以节约内存资源。

ps：整数集没有降级操作。

### skiplist 跳表
跳表是一种**多层有序**链表（多级索引），目的是加速对于有序数据的快速定位和范围查询，单个查询平均时间复杂度 O(logN)。
```c
typedef struct zskiplistNode {
    sds ele; // 元素值
    double score; // 元素权重
    struct zskiplistNode *backward; // 后向指针
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前向指针
        unsigned int span; // 跨度（离forward node的距离）
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;
    int level;
} zskiplist;
```

level数组保存了当前节点在每一层的下一个节点以及跨度（如果有），最底层 level0 是一个双向链表保存所有的节点，其他的层都相当于是索引。

![跳表图示](https://learn.lianglianglee.com/%e4%b8%93%e6%a0%8f/Redis%20%e6%ba%90%e7%a0%81%e5%89%96%e6%9e%90%e4%b8%8e%e5%ae%9e%e6%88%98/assets/35b2c22120952e1fac46147664e75b23-20221013234921-42s0bzg.jpg)

问：在新建节点时如何确定在哪层？
答：随机的。如果将每层节点数设置为下一层的一半会导致每次节点增删都可能需要调整level信息，带来额外的开销。

### radix tree 基数树
radix tree是前缀树（trie tree）的一种，相比于哈希表能更好的**节省内存空间**，于redis5.0引入。

radix tree包含两类节点，非压缩节点（单个字符）和压缩节点（多字符），非叶子节点**无法**同时指向这两种子节点。

叶子节点不包含字符，用于指向实际的值。

目的：快速匹配，范围查询，节约内存

应用：gin框架中的路由、golang中依靠radix tree对管理内存页的bitmap快速搜索
## 五种基本类型
### String
底层数据结构：int或SDS

常用命令：`SET GET SETNX SETEX MSET MGET INCR INCRBY EXPIRE DEL STRLEN`

应用场景：缓存对象（json作值）、计数、分布式锁、session

### List
底层数据结构：
* 3.2之前：元素个数和元素大小小于阈值时使用压缩列表ziplist，否则使用双向链表
* 3.2之后：quicklist

常用命令：`LPUSH LPOP LRANGE LINDEX BLPOP`

应用场景：消息队列（使用BRPOP可以阻塞读取，节省CPU开销）

ps：使用List作为消息队列存在缺陷，具体见[Stream](###Stream)

### Hash
底层数据结构：
* 7.0之前：元素个数和元素大小小于阈值时使用压缩列表ziplist，否则使用哈希表
* 7.0之后：使用了 listpack 替代压缩列表

常用命令：`HSET HMSET HDEL HLEN HGETALL HINCREBY HSCAN`

应用场景：缓存对象

ps1：用Hash做缓存相比于string的优势：便于维护、节省空间

ps2：如何使用ziplist存储的？将所有的`field-value`连续存储，列表结构类似：`[field1, value1, field2, value2]`

### Set
特性：无序、不可重复、支持多个集合取交集、并集、差集

底层数据结构：集合中的元素都是**整数**且元素个数小于阈值时使用整数集intset，否则使用哈希表

常用命令：`SADD SREM SMEMBERS SCARD SISMEMBER SRANDMEMBER SPOP SINTER SUNION SDIFF SSCAN`

应用场景：标签（tag）、点赞

ps：为什么选用intset而不用ziplist：插入时需保证唯一性，因而需要查找是否有相同元素，intset可以**二分**查找而ziplist只能遍历。

### Zset
特性：不可重复、可排序

底层数据结构：
* 7.0之前：元素个数和元素大小小于阈值时使用压缩列表ziplist，否则使用跳表skiplist
* 7.0之后：使用了 listpack 替代压缩列表

常用命令：`ZADD ZREM ZSCORE ZCARD ZRANGE ZREVRANGE ZRAN ZSCAN`

应用场景：排行榜

ps：如何使用ziplist存储的？将所有的`member-score`连续存储，列表结构类似：`{member, score}`

## 四种拓展类型
### BitMap
bitmap并不是一个真正的类型，而是基于String类型的面向位（bit-oriented）的操作。string类型可以当作一个位数组（bit vector）

特性：适用于**数据量大**的**二值统计**场景

底层原理：由于string是一个字节数组，一个字节8bit，利用bit串进行计数

常用命令：`SETBIT GETBIT BITCOUNT BITPOS BITOP [AND, OR, XOR and NOT]`

应用场景：签到统计、布隆过滤器

### HyperLogLog
特性：使用极小的内存对大量数据进行不精确的去重计数，存在误差

底层原理：通过`伯努利实验 + 极大似然估计`反推样本总数

常用命令：`PFADD PFCOUNT PFMERGE`

应用场景：网页UV计数

### GEO
特性：用于存储地理位置信息，并对存储的信息进行操作，包括计算两点距离以及获取区域内元素。

底层原理：直接使用zset，使用GeoHash编码方式将经纬度转换为Zset中的元素权重分数

常用命令：`GEOADD GEOPOS GEODIST GEORADIUS`

应用场景：滴滴打车

### Stream
在5.0之前，消息队列是使用list实现的，但是存在以下的几点问题：
1. 如果需要判断重复消息，生产者需要自行为每条消息生成一个全局唯一ID。
2. 消息可靠性无法保证，当消费者处理时出现宕机会导致当前消息丢失。可以创建一个备份list，但无法完全保证可靠。
3. 不支持多个消费者消费同一条消息
4. 不支持消费组的实现，消费组：组内一个成员读取了消息，同组其他成员就不能再读取同一个消息了。

stream的出现目的就是让消息队列更加稳定可靠。

特性：自动生成ID、支持ack确认消息的模式、支持消息持久化、支持消费组

底层数据结构：radix tree + listpack，radix树中key值为消息ID，value是listpack结构存储的具体数据。

常用命令：`XADD XLEN [XREAD STREAM] XDEL XRANGE XREADGROUP XPNDING XACK`

应用场景：消息队列、负载均衡

功能：
* 消息保序：XADD/XREAD
* 阻塞读取：XREAD block
* 重复消息处理：Stream 在使用 XADD 命令，会自动生成全局唯一 ID；
* 消息可靠性：内部使用 PENDING List 自动保存消息，使用 XPENDING 命令查看消费组已经读取但是未被确认的消息，消费者使用 XACK 确认消息；
* 支持消费组形式消费数据

缺点：
1. 可靠性：在以下 2 个场景下，都会导致数据丢失：
* AOF 持久化配置为每秒写盘，但这个写盘过程是异步的，Redis 宕机时会存在数据丢失的可能
* 主从复制也是异步的，主从切换时，也存在丢失数据的可能。

2. 可堆积：队列长度超过上限后，旧消息会被删除，只保留固定长度的新消息。

ps：发布/订阅机制为什么不可以作为消息队列？1.不具备数据持久化的能力（不会记录到日志文件）2.发后即忘 3.消息积压会导致消费端断开