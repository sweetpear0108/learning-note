# 持久化之rdb和混合快照
## 参考
[Redis 核心技术与实战](https://time.geekbang.org/column/intro/100056701)

[小林coding redis](https://www.xiaolincoding.com/redis/storage/rdb.html)

## 定义
RDB：Redis DataBase

RDB 记录的是某一时刻的数据，因此恢复效率会比aof方式高。

## 全量快照
记录某一时刻的全部数据
### 同步执行
通过`save`命令在主进程中生成rdb文件，写入期间会阻塞主进程。

问题：主进程可能长时间阻塞

### 异步执行
通过`bgsave`命令或者在配置文件中配置，由创建的子进程生成rdb文件，可以避免主进程阻塞。

同时，使用[写时复制技术]()保证了快照的完整性，也允许主线程同时对数据进行修改，避免了对正常业务的影响。但是，也导致主进程对数据的修改不会应用在本次的快照之中。

问题：
* 执行频率过高：频繁将全量数据写入磁盘，造成磁盘压力过大；另外，频繁fork子进程也会阻塞主进程。
* 执行频率过低：宕机会导致两次快照期间的数据丢失。

## 增量快照
做一次全量快照，后续每个快照时刻只需要记录增量修改的元数据信息。

问题：为了记录修改信息引入的额外空间开销较大。

## 混合快照
通过配置文件开启混合快照，使用rdb格式记录全量数据，同时使用aof格式记录两次快照间的增量记录，在下一次快照时清空aof记录。

优势：兼顾了rdb恢复速度快和aof数据少丢失的优点