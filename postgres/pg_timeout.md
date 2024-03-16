# Postgres 批量删除语句超时
## 场景
播放记录表`history`中大约有将近10亿条数据，为了减少内存占用，后台任务以每秒一次的速度删除超过1年未更新的旧记录。

* 表结构（示意结构）
```sql
        Table "public.history"
 Column |  Type  | Collation | Nullable | Default
--------+--------+-----------+----------+---------
 uid    | text   |           | not null |
 vid    | text   |           | not null |
 ts     | bigint |           | not null |
Indexes:
    "history_pkey" PRIMARY KEY, btree (uid, vid)
    "history_ts" btree (ts)
```

* 删除语句
`delete from history where (uid, vid) in (select uid, vid from history where ts <= $1 limit 500)`

## 问题出现
任务稳定运行了一年多的时间，突然有一天线上日志出现了大量的超时报错，排查后发现是pg执行删除语句的时候超过了最大时限 `pq: canceling statement due to user request`

## 问题排查
### 分析语句执行计划
`EXPLAIN ANALYZE statement`查看该语句的执行情况，发现长时间无响应，缩小`limit`数量重试，得到：

```bash
                                                                                     QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Delete on history  (cost=32.45..119.05 rows=83 width=87) (actual time=3791.173..3791.175 rows=0 loops=1)
   ->  Nested Loop  (cost=32.45..119.05 rows=83 width=87) (actual time=3774.474..3789.531 rows=10 loops=1)
         ->  HashAggregate  (cost=31.75..31.85 rows=10 width=138) (actual time=3771.885..3771.904 rows=10 loops=1)
               Group Key: "ANY_subquery".uid, "ANY_subquery".vid
               Batches: 1  Memory Usage: 24kB
               ->  Subquery Scan on "ANY_subquery"  (cost=0.00..31.70 rows=10 width=138) (actual time=103.109..3771.850 rows=10 loops=1)
                     ->  Limit  (cost=0.00..31.60 rows=10 width=57) (actual time=103.104..3771.828 rows=10 loops=1)
                           ->  Seq Scan on history history_1  (cost=0.00..48477444.80 rows=15343146 width=57) (actual time=103.103..3771.821 rows=10 loops=1)
                                 Filter: (ts <= '1678603902000'::bigint)
                                 Rows Removed by Filter: 6974992
         ->  Index Scan using history_pkey on history  (cost=0.70..8.72 rows=1 width=63) (actual time=1.757..1.757 rows=1 loops=10)
               Index Cond: ((uid = "ANY_subquery".uid) AND (vid = "ANY_subquery".vid))
 Planning Time: 0.376 ms
 Execution Time: 3791.369 ms
(14 行记录)
```
发现主要阻塞在子查询，优化器竟然选择了`Seq Scan`顺序扫描，大量不符合条件的数据被`Filter`筛掉，时间就耗在这个上边了。

`啊？ts上不是有索引么，这么大的表竟然选择了顺序扫描？`

### 问一下GPT:顺序和索引扫描是如何取舍的？

在PostgreSQL中，即使在选择相关列上存在索引，查询计划也可能选择顺序扫描（Sequential Scan）而不是索引扫描（Index Scan）。这是因为查询优化器需要基于多种因素做出决策，如预计的查询代价，表大小，数据的分布等。以下是一些可能的原因：

1. **全表扫描优于索引扫描**：倘若你要查询的数据占据了表的大部分，顺序扫描可能更有效。PostgreSQL查询优化器会权衡全表扫描和索引扫描的代价，如果全表扫描的预计代价更低，那么它可能会选择全表扫描。

2. **统计信息不准确**：PostgreSQL的查询优化器基于统计信息对查询计划进行优化。如果表的统计信息不准确（例如，表数据已经发生了大幅度变化但是没有重新收集统计信息），查询优化器可能会做出错误的决策，即使索引扫描更高效，也可能选择顺序扫描。

3. **索引列的选择性不高**：如果索引列的值重复度很高（例如，性别、是否启用等值只有几种的列），那么使用索引扫描的效率可能并不高，查询优化器可能会选择顺序扫描。

4. **系统参数设置的问题**：某些PostgreSQL系统参数可能影响到索引的使用，比如`enable_seqscan`参数控制了是否优先使用顺序扫描。

如果你注意到了查询计划中一直使用序列扫描而不是索引扫描，且索引扫描明显更高效，你可能需要重新收集统计信息（使用`ANALYZE`命令），或者检查你的查询是否能够有效的利用索引，或者审查并调整相关的系统参数。

### 逐一排查
1. 全表扫描优于索引扫描？不确定。全表10亿条数据，全表扫描会比索引扫描更高效么？

2. 统计信息不准确？不确定。

3. 索引列的选择性不高？不太可能🙅。ts是按照观看时间排序的，不存在重复度很高的情况。

4. 系统参数设置的问题？不太可能🙅。任务正常运行一段时间了。

ps：`EXPLAIN`中cost代表查询优化器估算的查询代价，原因一关于全表扫描和索引扫描的取舍就取决于优化器对两种扫描方式的**代价估算**，而估算则需要参考原因二提到的**统计信息**，也就是说一二是相关的。

### 有索引但不走的情况有哪些？
1. 表很小（100行以内），顺序扫描的效率可能会高于索引扫描。

2. 返回结果占总表很大比例。

3. 当`LIMIT`数量很小时，优化器可能会认为“顺序扫描直到结果达到限制数量后终止”会更快的找到结果；但这是**双刃剑**，依赖于表统计数据的准确性。

### 优化器参考的统计信息有哪些？

1. **表的大小**：一般来说，对于非常小的表，顺序扫描往往更有效，因为索引扫描涉及到查找索引并获取磁盘上数据的过程，这对于小表来说可能比直接顺序扫描更耗时。相反，对于大表，索引扫描往往能显著提高效率。

2. **索引的选择性**：索引的选择性是指通过索引能够区分的记录的百分比。如果索引的选择性较高（例如，主键索引），邐那么索引扫描往往更有效。

3. **查询的选择度**：如果查询过滤大部分数据，那么索引扫描对于性能的提升更明显。因为顺序扫描需要扫描整个表，而索引扫描只需要访问满足条件的索引项和相关记录。

4. **硬件性能**：硬件的 IO 性能和 CPU 性能也会影响扫描方式的选择。IO 密集的工作负载可能更适合使用索引扫描，而 CPU 密集的工作负载可能更适合使用顺序扫描。

5. **索引和表的物理顺序**：如果表的物理顺序与索引的顺序高度相关（correlation 接近 1），那么使用索引扫描的成本可能降低，因为顺序 IO 往往比随机 IO 更高效。

## 问题定位
通过排查可以猜测：是优化器在执行查询命令时，对代价估算出现了错误，导致选择了实际性能更差的全表扫描方法。

### 横向对比其他表
除了`history`表之外，数据库中还有两个结构相似、数据量相似、索引设置相似的记录表`history_a`,`history_b`（简称a表b表），其中a表和history一样在执行后台删除任务，b表没有执行过删除任务。

浅看一眼表内总行数：
```bash
x=> SELECT relname, reltuples FROM pg_class r JOIN pg_namespace n ON (relnamespace = n.oid) WHERE relkind = 'r' AND n.nspname = 'public';
       relname     |   reltuples
-------------------+---------------
 history_a         |  7.233728e+08
 history           | 9.3278854e+08
 history_b         |  8.412001e+08
```

对history、a、b分别执行查询命令：
```bash
x=> explain analyze select uid, vid from history_a where ts <= 1678692987000 limit 100;
                                                                        QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.57..329.24 rows=100 width=57) (actual time=0.074..11.583 rows=100 loops=1)
   ->  Index Scan using history_a_ts_1 on history_a  (cost=0.57..49428562.93 rows=15039219 width=57) (actual time=0.073..11.573 rows=100 loops=1)
         Index Cond: (ts <= '1678692987000'::bigint)
 Planning Time: 1.400 ms
 Execution Time: 11.617 ms
(5 行记录)

x=> explain analyze select uid, vid from history where ts <= 1678692987000 limit 100;
                                                               QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..262.93 rows=100 width=57) (actual time=519.878..14138.338 rows=100 loops=1)
   ->  Seq Scan on history  (cost=0.00..48477444.80 rows=18437250 width=57) (actual time=519.877..14138.275 rows=100 loops=1)
         Filter: (ts <= '1678692987000'::bigint)
         Rows Removed by Filter: 24919480
 Planning Time: 0.671 ms
 Execution Time: 14138.486 ms
(6 行记录)

x=> explain analyze select uid, vid from history_b where ts <= 1678692987000 limit 100;
                                                                  QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..14.53 rows=100 width=77) (actual time=2.032..12.098 rows=100 loops=1)
   ->  Seq Scan on history_b  (cost=0.00..42239628.56 rows=290642848 width=77) (actual time=2.031..12.085 rows=100 loops=1)
         Filter: (ts <= '1678692987000'::bigint)
         Rows Removed by Filter: 1033
 Planning Time: 0.610 ms
 Execution Time: 15.866 ms
(6 行记录)
```

* 好奇怪，发现a表选择了索引扫描，b和history表都是顺序扫描，但是b表执行速度显著快于history表。

### 纵向对比自己
```bash
x=> explain analyze select uid, vid from history where ts <= 1678763273000 limit 500;
                                                              QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..1166.31 rows=500 width=57) (actual time=10.661..7664.132 rows=500 loops=1)
   ->  Seq Scan on history  (cost=0.00..48421139.20 rows=20758242 width=57) (actual time=10.660..7663.903 rows=500 loops=1)
         Filter: (ts <= '1678763273000'::bigint)
         Rows Removed by Filter: 14947898
 Planning Time: 0.056 ms
 Execution Time: 7666.074 ms
(6 行记录)

x=> explain analyze select uid, vid from history where ts <= 1678763273000 order by ts limit 500;
                                                                             QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.57..1595.46 rows=500 width=65) (actual time=1.700..64.891 rows=500 loops=1)
   ->  Index Scan using history_ts_1 on history  (cost=0.57..66214154.35 rows=20758242 width=65) (actual time=1.699..64.824 rows=500 loops=1)
         Index Cond: (ts <= '1678763273000'::bigint)
 Planning Time: 0.053 ms
 Execution Time: 64.949 ms
(5 行记录)
```

* 添加了`order by`强制命令走索引，发现走索引的估算cost比顺序扫描的估算cost要低，说明了优化器为什么会选择顺序扫描。同样也说明**估算出现了问题**。

### 猜想
1. 批量删除导致了顺序扫描耗时增加，可能和频繁删除插入导致磁盘物理顺序乱序有关，因此在遍历时需要访问更多的内存页。
2. 优化器在history表上决策执行计划时**参考的统计数据有问题**，导致代价估算出现偏差，可能的原因是没有及时更新统计数据。

### 验证
查询history表和a表的状态信息表，查看最后执行`vacuum`和`analyze`的时间。

```bash
x=> select relname,last_analyze,last_autoanalyze,last_autovacuum from  pg_stat_user_tables where relname='history_a' or relname='history';
-[ RECORD 1 ]----+------------------------------
relname          | history_a
last_analyze     |
last_autoanalyze | 2024-03-13 07:05:50.585392+00
last_autovacuum  | 2024-03-11 13:13:25.05856+00
-[ RECORD 2 ]----+------------------------------
relname          | history
last_analyze     |
last_autoanalyze |
last_autovacuum  | 2024-03-13 16:54:04.673357+00
```

* history表竟然没有配置`auto analyze`。。。。。。（b表也没配）

## 问题解决
### 手动执行 ANALYZE
`analyze history`会以**采样**的方式对表统计数据进行估算，所以速度相对较快；并且只会阻塞如 vacuum、创建索引、reindex、alter table等命令，**不会影响表的读写**。

### 重新执行命令
```bash
x=> explain analyze select uid, vid from history where ts <= 1678795619000 limit 500;
                                                                           QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.57..1978.10 rows=500 width=57) (actual time=11.670..16.685 rows=500 loops=1)
   ->  Index Scan using history_ts_1 on history  (cost=0.57..495456.97 rows=125272 width=57) (actual time=11.669..16.638 rows=500 loops=1)
         Index Cond: (ts <= '1678795619000'::bigint)
 Planning Time: 11.769 ms
 Execution Time: 17.972 ms
(5 行记录)
```

**问题解决。**

此外，还可以修改limit值观察一下是不是当limit很小时会顺序查询：
```bash
x=> explain analyze select uid, vid from history where ts <= 1710418557000 limit 1;
                                                            QUERY PLAN
----------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.00..0.05 rows=1 width=57) (actual time=1.055..1.056 rows=1 loops=1)
   ->  Seq Scan on history  (cost=0.00..48494915.20 rows=924846115 width=57) (actual time=1.054..1.054 rows=1 loops=1)
         Filter: (ts <= '1710418557000'::bigint)
 Planning Time: 6.117 ms
 Execution Time: 1.076 ms
(5 行记录)
```

果然如此。

## 索引优化
在寻找问题的过程中，发现索引内存占用较大，可能与数据频繁的删除新增有关，称之为**索引膨胀**

查询数据库中所有表的大小：
```sql
SELECT table_name,
       pg_size_pretty(table_size)   AS table_size,
       pg_size_pretty(indexes_size) AS indexes_size,
       pg_size_pretty(total_size)   AS total_size
FROM (
         SELECT table_name,
                pg_table_size(table_name)          AS table_size,
                pg_indexes_size(table_name)        AS indexes_size,
                pg_total_relation_size(table_name) AS total_size
         FROM (
                  SELECT ('"' || table_schema || '"."' || table_name || '"') AS table_name
                  FROM information_schema.tables
                  where table_schema = 'public'  
              ) AS all_tables
         ORDER BY total_size DESC
     ) AS pretty_sizes;
```

reindex:
```sql
REINDEX TABLE CONCURRENTLY history;
```

或者手动

```sql
CREATE INDEX CONCURRENTLY history_ts_1 ON history(ts);
DROP INDEX CONCURRENTLY history_ts;

CREATE UNIQUE INDEX CONCURRENTLY history_pkey_new ON history(uid, vid);
ALTER TABLE history DROP CONSTRAINT history_pkey, ADD CONSTRAINT history_pkey PRIMARY KEY USING INDEX history_pkey_new;
```