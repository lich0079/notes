
# MySQL实战宝典

## 整形

* 自增

* 用bigint 做主键  ， int只有42亿

* 8.0前会回溯

* 推荐用big int 来存 金融数据，计算效率高

* 不推荐使用unsigned, 数据分析（++--）的时候结果并不是想要得到的结果。

* 不用float double，后续版本不支持





## 字符串

- char(N)  0 ~255
- N 是字符  不是字节 不同字符集 字节数不一样， char本质也是变长
- varchar(N) 0 ~65536
- 更大用 text  blob



字符集 

GBK UTF8  推荐 UTF8MB4（有更多表情字符）

```
[mysqld]
character-set-server = utf8mb4
```

Collation 排序规则

```
mysql> SHOW CHARSET LIKE 'utf8%';
+---------+---------------+--------------------+--------+
| Charset | Description   | Default collation  | Maxlen |
+---------+---------------+--------------------+--------+
| utf8    | UTF-8 Unicode | utf8_general_ci    |      3 |
| utf8mb4 | UTF-8 Unicode | utf8mb4_0900_ai_ci |      4 |
+---------+---------------+--------------------+--------+
2 rows in set (0.01 sec)
```

排序规则以 _ci 结尾，表示不区分大小写（Case Insentive），_cs 表示大小写敏感，_bin 表示通过存储字符的二进制进行比较



正确修改列字符集的命令应该使用 ALTER TABLE ... CONVERT TO...

```
mysql> ALTER TABLE emoji_test CONVERT TO CHARSET utf8mb4;
```



```
CREATE TABLE `xxx` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `symbol` int(11) NOT NULL DEFAULT '0' COMMENT 'symbol id',

  PRIMARY KEY (`id`) USING BTREE,
  UNIQUE KEY `symbol_period_start_at_unique` (`symbol`,`period`,`start_at`)
) ENGINE=InnoDB AUTO_INCREMENT=64875 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci ROW_FORMAT=COMPACT;
```





## 日期类型

- 类型 DATETIME 最终展现的形式为：YYYY-MM-DD HH：MM：SS，固定占用 8 个字节    datetime好像是占5个字节，datetime(6)占8个字节
- TIMESTAMP  时间戳类型，其实际存储的内容为‘1970-01-01 00:00:00’到现在的毫秒数。在 MySQL 中，由于类型 TIMESTAMP 占用 4 个字节，因此其存储的时间上限只能到‘2038-01-19 03:14:07’
- 若带有毫秒时，类型 TIMESTAMP 占用 7 个字节，而 DATETIME 无论是否存储毫秒信息，都占用 8 个字节
- TIMESTAMP 日期存储的上限为 2038-01-19 03:14:07，业务用 TIMESTAMP 存在风险
- 使用 TIMESTAMP 必须显式地设置时区，不要使用默认系统时区，否则存在性能问题，推荐在配置文件中设置参数 time_zone = '+08:00'；
- DATETIME vs TIMESTAMP vs INT,   建议你使用类型 DATETIME

```
# 比较time_zone为System和Asia/Shanghai的性能对比

mysqlslap -uroot --number-of-queries=1000000 --concurrency=100 --query='SELECT NOW()'
```




## Json

5.7 版本开始支持的功能，而 8.0 版本解决了更新 JSON 的日志性能瓶颈， 建议8.0

JSON 类型比较适合存储一些修改较少、相对静态的数据



JSON_EXTRACT

```
SELECT
    userId,
    JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.cellphone")) cellphone,
    JSON_UNQUOTE(JSON_EXTRACT(loginInfo,"$.wxchat")) wxchat
FROM UserLogin;
```

写 JSON_EXTRACT、JSON_UNQUOTE 非常麻烦，MySQL 还提供了 ->> 表达式

```
SELECT
    userId,
    loginInfo->>"$.cellphone" cellphone,
    loginInfo->>"$.wxchat" wxchat
FROM UserLogin;
```

json 索引

```
// SQL 首先创建了一个虚拟列 cellphone，这个列是由函数 loginInfo->>"$.cellphone" 计算得到的。然后在这个虚拟列上创建一个唯一索引 idx_cellphone
ALTER TABLE UserLogin ADD COLUMN cellphone VARCHAR(255) AS (loginInfo->>"$.cellphone");

ALTER TABLE UserLogin ADD UNIQUE INDEX idx_cellphone(cellphone);
```

Multi-Valued Indexes，用于在 JSON 数组上创建索引

```
INSERT INTO UserTag VALUES (1,'[2,6,8,10]');
INSERT INTO UserTag VALUES (2,'[3,10,12]');

ALTER TABLE UserTag
ADD INDEX idx_user_tags ((cast((userTags->"$") as unsigned array)));
```

函数  member of、json_contains、json_overlaps  

```
SELECT * FROM UserTag 
WHERE 10 MEMBER OF(userTags->"$")
```







## 主键设计

- 每张表一定要有一个主键；

- 自增主键只推荐用在非核心业务表，甚至应避免使用；

    自增存在回溯问题；

    自增值在服务器端产生，存在并发性能问题；

    自增值做主键，只能在当前实例中保证唯一，不能保证全局唯一；

    公开数据值，容易引发安全问题，例如知道地址http://www.example.com/User/10/，很容猜出 User 有 11、12 依次类推的值，容易引发数据泄露；

    MGR（MySQL Group Replication） 可能引起的性能问题；

    分布式架构设计问题。

- 核心业务表推荐使用 UUID 或业务自定义主键；

##    



## 表压缩

ROW_FORMAT 设置为 COMPRESS，表示启用 COMPRESS 页压缩功能，KEY_BLOCK_SIZE 设置为 8，表示将一个 16K 的页压缩为 8K

磁盘少了， 内存占用多了

COMPRESSION=ZLIB        TPC（Transparent Page Compression）

- MySQL 中的压缩都是基于页的压缩；

- COMPRESS 页压缩适合用于性能要求不高的业务表，如日志、监控、告警表等；

- COMPRESS 页压缩内存缓冲池存在压缩和解压的两个页，会严重影响性能；

- 对存储有压缩需求，又希望性能不要有明显退化，推荐使用 TPC 压缩；

- 通过 ALTER TABLE 启用 TPC 压缩后，还需要执行命令 OPTIMIZE TABLE 才能立即完成空间的压缩。

  

## 索引

B+树索引的特点是： 基于磁盘的平衡树，但树非常矮，通常为 3~4 层，能存放千万到上亿的排序数据。树矮意味着访问效率高，从千万或上亿数据里查询一条数据，只用 3、4 次 I/O。

```
在 MySQL InnoDB 存储引擎中，一个页的大小为 16K，在上面的表 User 中，键值 userId 是BIGINT 类型，则：


根节点能最多存放以下多个键值对 = 16K / 键值对大小(8+6) ≈ 1100

再假设表 User 中，每条记录的大小为 500 字节，则：
叶子节点能存放的最多记录为 = 16K / 每条记录大小 ≈ 32

综上所述，树高度为 2 的 B+ 树索引，最多能存放的记录数为：
总记录数 = 1100 * 32 =  35,200


树高度为 3 的 B+ 树索引
总记录数 = 1100（根节点） * 1100（中间节点） * 32 =  38,720,000
```

UUID 由于是无序值，所以在插入时性能比起顺序值自增 ID 和排序 UUID，性能上差距比较明显。

**所以，我再次强调：** 在表结构设计时，主键的设计一定要尽可能地使用顺序值，这样才能保证在海量并发业务场景下的性能。



通过查询表 mysql.innodb_index_stats 查看每个索引的大致情况

```
SELECT 
table_name,index_name,stat_name,
stat_value,stat_description 
FROM innodb_index_stats 
WHERE table_name = 'orders' and index_name = 'PRIMARY';

+----------+------------+-----------+------------+------------------+
|table_name| index_name | stat_name | stat_value |stat_description  |
+----------+-------------------+------------+------------+----------+
| orders | PRIMARY|n_diff_pfx01|5778522     | O_ORDERKEY            |
| orders | PRIMARY|n_leaf_pages|48867 | Number of leaf pages        |
| orders | PRIMARY|size        |49024 | Number of pages in the index|
+--------+--------+------------+------+-----------------------------+
3 rows in set (0.00 sec)
```

怎么知道哪些 B+树索引未被使用过呢

```
SELECT * FROM schema_unused_indexes
WHERE object_schema != 'performance_schema';

+---------------+-------------+--------------+
| object_schema | object_name | index_name   |
+---------------+-------------+--------------+
| sbtest        | sbtest1     | k_1          |
| sbtest        | sbtest2     | k_2          |
| sbtest        | sbtest3     | k_3          |
| sbtest        | sbtest4     | k_4          |
| tpch          | customer    | CUSTOMER_FK1 |
| tpch          | lineitem    | LINEITEM_FK2 |
| tpch          | nation      | NATION_FK1   |
| tpch          | orders      | ORDERS_FK1   |
| tpch          | partsupp    | PARTSUPP_FK1 |
| tpch          | supplier    | SUPPLIER_FK1 |
+---------------+-------------+--------------+
```

MySQL 8.0 版本推出了索引不可见（Invisible）功能。在删除废弃索引前，用户可以将索引设置为对优化器不可见，然后观察业务是否有影响。若无，DBA 可以更安心地删除这些索引

```
ALTER TABLE t1 
ALTER INDEX idx_name INVISIBLE/VISIBLE;
```



## 存储

堆表：  数据 和 索引分开存储

索引组织表（Index Organized Table） 也叫聚集索引（Clustered Index）    数据都在索引叶子节点

**二级索引（Secondeary Index）**

叶子节点存放的是索引键值、主键值

通过二级索引 idx_name 只能定位主键值，需要额外再通过主键索引进行查询，才能得到最终的结果。**这种“二级索引通过主键索引进行再一次查询”的操作叫作“回表”  （避免select \*）**

**函数索引**

优化业务 SQL 性能

配合虚拟列（Generated Column）

```jsx
ALTER TABLE User 
ADD INDEX 
idx_func_register_date((DATE_FORMAT(register_date,'%Y-%m')));
```

虚拟列不占用实际存储空间，在虚拟列上创建索引本质就是函数索引。





## 组合索引

- 覆盖多个查询条件，如（a，b）索引可以覆盖查询 a = ? 或者 a = ? and b = ?；  但不能覆盖 b = ?
- 避免 SQL 的额外排序，提升 SQL 性能，如 WHERE a = ? ORDER BY b 这样的查询条件；
- 利用组合索引包含多个列的特性，可以实现索引覆盖技术，提升 SQL 的查询性能，用好索引覆盖技术，性能提升 10 倍不是难事, 避免回表









## CBO

（Cost-based Optimizer)

MySQL 数据库由 Server 层和 Engine 层组成：

- Server 层有 SQL 分析器、SQL优化器、SQL 执行器，用于负责 SQL 语句的具体执行过程；
- Engine 层负责存储具体的数据，如最常使用的 InnoDB 存储引擎，还有用于在内存中存储临时结果集的 TempTable 引擎。

```jsx
Cost  = Server Cost + Engine Cost
      = CPU Cost + IO Cost
select * from mysql.server_cost

select * from mysql.engine_cost
```

表 server_cost 记录了 Server 层优化器各种操作的成本，这里面包括了所有 CPU Cost，其具体含义如下。

- disk_temptable_create_cost：创建磁盘临时表的成本，默认为20。
- disk_temptable_row_cost：磁盘临时表中每条记录的成本，默认为0.5。
- key_compare_cost：索引键值比较的成本，默认为0.05，成本最小。
- memory_temptable_create_cost：创建内存临时表的成本：默认为1。
- memory_temptable_row_cost：内存临时表中每条记录的成本，默认为0.1。
- row_evaluate_cost：记录间的比较成本，默认为0.1。

表 engine_cost 记录了存储引擎层各种操作的成本，这里包含了所有的 IO Cost，具体含义如下。

- io_block_read_cost：从磁盘读取一个页的成本，默认值为1。
- memory_block_read_cost：从内存读取一个页的成本，默认值为0.25。

```jsx
// 通过命令 EXPLAIN的FORMAT=json 来查看各成本的值

EXPLAIN FORMAT=json 
SELECT o_custkey,SUM(o_totalprice) 
FROM orders GROUP BY o_custkey
```

B+ 树索引通常要建立在高选择性的字段或字段组合上，如性别、订单 ID、日期等，因为这样每个字段值大多并不相同。

- 一般只对高选择度的字段和字段组合创建索引，低选择度的字段如性别，不创建索引；
- 低选择性，但是数据存在倾斜，通过索引找出少部分数据，可以考虑创建索引；
- 若数据存在倾斜，可以创建直方图，让优化器知道索引中数据的分布，进一步校准执行计划。



## Join

MySQL 数据库中支持 JOIN 连接的算法有 Nested Loop Join 和 Hash Join 两种，前者通常用于 OLTP 业务，后者用于 OLAP 业务。在 OLTP 可以写 JOIN，优化器会自动选择最优的执行计划。但若使用 JOIN，要确保 SQL 的执行计划使用了正确的索引以及索引覆盖，



### Nested Loop Join

以下三种 JOIN 类型，驱动表各是哪张表：

```
SELECT ... FROM R LEFT JOIN S ON R.x = S.x WEHRE ...

SELECT ... FROM R RIGHT JOIN S ON R.x = S.x WEHRE ...

SELECT ... FROM R INNER JOIN S ON R.x = S.x WEHRE ...
```

对于上述 Left Join 来说，驱动表就是左表 R；Right Join中，驱动表就是右表 S。这是 JOIN 类型决定左表或右表的数据一定要进行查询。但对于 INNER JOIN，驱动表可能是表 R，也可能是表 S。

在这种场景下，**谁需要查询的数据量越少，谁就是驱动表**

优化器一般认为，通过索引进行查询的效率都一样，所以 Nested Loop Join 算法主要要求驱动表的数量要尽可能少

### Hash Join

Hash Join会扫描关联的两张表：

- 首先会在扫描驱动表的过程中创建一张哈希表；
- 接着扫描第二张表时，会在哈希表中搜索每条关联的记录，如果找到就返回记录。



## 子查询

子查询 IN 和 EXISTS，哪个性能更好?    关注 SQL 执行计划

- 子查询相比 JOIN 更易于人类理解，所以受众更广，使用更多；
- 当前 MySQL 8.0 版本可以“毫无顾忌”地写子查询，对于子查询的优化已经相当完备；
- 对于老版本的 MySQL，**请 Review 所有子查询的SQL执行计划，** 对于出现 DEPENDENT SUBQUERY 的提示，请务必即使进行优化，否则对业务将造成重大的性能影响；
- DEPENDENT SUBQUERY 的优化，一般是重写为派生表进行表连接。表连接的优化就是我们12讲所讲述的内容



## 分区表

分区表就是把**物理表结构相同的几张表，通过一定算法，组成一张逻辑大表**。这种算法叫“分区函数”，当前 MySQL 数据库支持的分区函数类型有 RANGE、LIST、HASH、KEY、COLUMNS。

分区表技术不是用于提升 MySQL 数据库的性能，而是方便数据的管理。**更多的是解决数据迁移和备份的问题**

主键也必须是分区列的一部分，不然创建分区表时会失败

```
CREATE TABLE t (
    a INT,
    b INT,
    c DATETIME,
    d VARCHAR(32),
    e INT,
    PRIMARY KEY (a,b,c),
    KEY idx_e (e)
)
partition by range columns(c) (
    PARTITION p0000 VALUES LESS THAN ('2019-01-01'),
    PARTITION p2019 VALUES LESS THAN ('2020-01-01'),
    PARTITION p2020 VALUES LESS THAN ('2021-01-01'),
    PARTITION p9999 VALUES LESS THAN (MAXVALUE)
);
```



分区表的索引都是局部，而非全局。也就是说，索引在每个分区文件中都是独立的，所以分区表上的唯一索引必须包含分区列信息，否则创建会报错

但正因为唯一索引包含了分区列，唯一索引也就变成仅在当前分区唯一，而不是全局唯一了



从表格中可以看到，B+ 树的高度为 4 能存放数十亿的数据，一次查询只需要占用 4 次 I/O，速度非常快。

但是当你使用分区之后，效果就不一样了，比如上面的表 t，我们根据时间拆成每年一张表，这时，虽然 B+ 树的高度从 4 降为了 3，但是这个提升微乎其微。

分区表还会引入新的性能问题，比如非分区列的查询。非分区列的查询，即使分区列上已经创建了索引，但因为索引是每个分区文件对应的本地索引，所以要查询每个分区,   上述 SQL 需要访问 4 个分区，假设每个分区需要 3 次 I/O，则这条 SQL 总共要 12 次 I/O。但是，如果使用普通表，记录数再多，也就 4 次的 I/O 的时间。

列子

```
CREATE TABLE `orders` (

  `o_orderkey` int NOT NULL,

  `O_CUSTKEY` int NOT NULL,

  `O_ORDERSTATUS` char(1) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,

  `O_TOTALPRICE` decimal(15,2) NOT NULL,

  `O_ORDERDATE` date NOT NULL,

  `O_ORDERPRIORITY` char(15) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,

  `O_CLERK` char(15) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,

  `O_SHIPPRIORITY` int NOT NULL,

  `O_COMMENT` varchar(79) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL,

  PRIMARY KEY (`o_orderkey`,`O_ORDERDATE`),

  KEY `orders_fk1` (`O_CUSTKEY`),

  KEY `idx_orderdate` (`O_ORDERDATE`)

)

PARTITION BY RANGE  COLUMNS(o_orderdate)

(

  PARTITION p0000 VALUES LESS THAN ('1992-01-01') ENGINE = InnoDB,

  PARTITION p1992 VALUES LESS THAN ('1993-01-01') ENGINE = InnoDB,

  PARTITION p1993 VALUES LESS THAN ('1994-01-01') ENGINE = InnoDB,

  PARTITION p1994 VALUES LESS THAN ('1995-01-01') ENGINE = InnoDB,

  PARTITION p1995 VALUES LESS THAN ('1996-01-01') ENGINE = InnoDB,

  PARTITION p1996 VALUES LESS THAN ('1997-01-01') ENGINE = InnoDB,

  PARTITION p1997 VALUES LESS THAN ('1998-01-01') ENGINE = InnoDB,

  PARTITION p1998 VALUES LESS THAN ('1999-01-01') ENGINE = InnoDB,

  PARTITION p9999 VALUES LESS THAN (MAXVALUE)

）

```

这条 SQL 的执行相当慢，产生大量二进制日志，在生产系统上，也会导致数据库主从延迟的问题

```
DELETE FROM Orders 
WHERE o_orderdate >= '1998-01-01' 
  AND o_orderdate < '1999-01-01'
```

使用分区表的话，对于数据的管理就容易多了，你直接使用清空分区的命令

```
ALTER TABLE orders_par 
TRUNCATE PARTITION p1998
```

上述 SQL 执行速度非常快，因为实际执行过程是把分区文件删除和重建。另外产生的日志也只有一条 DDL 日志，也不会导致主从复制延迟问题。



## 主从复制

数据库复制本质上就是数据同步。MySQL 数据库是基于二进制日志（binary log）进行数据增量同步，而二进制日志记录了所有对于 MySQL 数据库的修改操作。

在默认 ROW 格式二进制日志中，一条 SQL 操作影响的记录会被全部记录下来，比如一条 SQL语句更新了三行记录，在二进制日志中会记录被修改的这三条记录的前项（before image）和后项（after image）。

对于 INSERT 或 DELETE 操作，则会记录这条被插入或删除记录所有列的信息

```
SHOW BINLOG EVENTS in 'binlog.000004'
```

- Master 服务器会把数据变更产生的二进制日志通过 Dump 线程发送给 Slave 服务器；
- Slave 服务器中的 I/O 线程负责接受二进制日志，并保存为中继日志；
- SQL/Worker 线程负责并行执行中继日志，即在 Slave 服务器上回放 Master 产生的日志。



### MySQL 复制配置

搭建 MySQL 复制实现非常简单，基本步骤如下：

1. 创建复制所需的账号和权限；
2. 从 Master 服务器拷贝一份数据，可以使用逻辑备份工具 mysqldump、mysqlpump，或物理备份工具 Clone Plugin；
3. 通过命令 CHANGE MASTER TO 搭建复制关系；
4. 通过命令 SHOW SLAVE STATUS 观察复制状态。

虽然 MySQL 复制原理和实施非常简单，但在配置时却容易出错，请你务必在配置文件中设置如下配置：

复制代码

```
gtid_mode = on

enforce_gtid_consistency = 1

binlog_gtid_simple_recovery = 1

relay_log_recovery = ON

master_info_repository = TABLE 

relay_log_info_repository = TABLE
```

上述设置都是用于保证 crash safe，即无论 Master 还是 Slave 宕机，当它们恢复后，连上主机后，主从数据依然一致，不会产生任何不一致的问题。

我经常听有同学反馈：MySQL会存在主从数据不一致的情况，**请确认上述参数都已配置，否则任何的不一致都不是 MySQL 的问题，而是你使用 MySQL 的错误姿势所致**。





### MySQL复制类型及应用选项

#### 异步复制

在异步复制（async replication）中，Master 不用关心 Slave 是否接收到二进制日志，所以 Master 与 Slave 没有任何的依赖关系。你可以认为 Master 和 Slave 是分别独自工作的两台服务器，数据最终会通过二进制日志达到一致。

异步复制的性能最好，因为它对数据库本身几乎没有任何开销，除非主从延迟非常大，Dump Thread 需要读取大量二进制日志文件。

如果业务对于数据一致性要求不高，当发生故障时，能容忍数据的丢失，甚至大量的丢失，推荐用异步复制，这样性能最好（比如像微博这样的业务，虽然它对性能的要求极高，但对于数据丢失，通常可以容忍）。但往往核心业务系统最关心的就是数据安全，比如监控业务、告警系统。

#### 半同步复制

半同步复制要求 Master 事务提交过程中，至少有 N 个 Slave 接收到二进制日志，这样就能保证当 Master 发生宕机，至少有 N 台 Slave 服务器中的数据是完整的。

半同步复制并不是 MySQL 内置的功能，而是要安装半同步插件，并启用半同步复制功能，设置 N 个 Slave 接受二进制日志成功，比如：

复制代码

```
plugin-load="rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"

rpl-semi-sync-master-enabled = 1

rpl-semi-sync-slave-enabled = 1

rpl_semi_sync_master_wait_no_slave = 1
```

上面的配置中：

- 第 1 行要求数据库启动时安装半同步插件；
- 第 2、3 行表示分别启用半同步 Master 和半同步 Slave 插件；
- 第 4 行表示半同步复制过程中，提交的事务必须至少有一个 Slave 接收到二进制日志。

在半同步复制中，有损半同步复制是 MySQL 5.7 版本前的半同步复制机制，这种半同步复制在Master 发生宕机时，**Slave 会丢失最后一批提交的数据**，若这时 Slave 提升（Failover）为Master，可能会发生已经提交的事情不见了，发生了回滚的情况。

有损半同步：    先commit 在wait ack

无损半同步：     先wait ack 在commit

#### 多源复制

无论是异步复制还是半同步复制，都是 1 个 Master 对应 N 个 Slave。其实 MySQL 也支持 N 个 Master 对应 1 个 Slave，这种架构就称之为多源复制。

上图显示了订单库、库存库、供应商库，通过多源复制同步到了一台 MySQL 实例上，接着就可以通过 MySQL 8.0 提供的复杂 SQL 能力，对业务进行深度的数据分析和挖掘。

#### 延迟复制

前面介绍的复制架构，Slave 在接收二进制日志后会尽可能快地回放日志，这样是为了避免主从之间出现延迟。而延迟复制却允许Slave 延迟回放接收到的二进制日志，为了避免主服务器上的误操作，马上又同步到了从服务器，导致数据完全丢失。

我们可以通过以下命令设置延迟复制：

复制代码

```
CHANGE MASTER TO master_delay = 3600
```

这样就人为设置了 Slave 落后 Master 服务器1个小时。

延迟复制在数据库的备份架构设计中非常常见，比如可以设置一个延迟一天的延迟备机，这样本质上说，用户可以有 1 份 24 小时前的快照。

那么当线上发生误操作，如 DROP TABLE、DROP DATABASE 这样灾难性的命令时，用户有一个 24 小时前的快照，数据可以快速恢复。

对金融行业来说，延迟复制是你备份设计中，必须考虑的一个架构部分。





###  逻辑日志的优缺点

假设有个 DELETE 删除操作，删除当月数据，由于数据量可能有 1 亿条记录，可能会产生 100G 的二进制日志，则这条 SQL 在提交时需要等待 100G 的二进制日志写入磁盘，如果二进制日志磁盘每秒写入速度为 100M/秒，至少要等待 1000 秒才能完成这个事务的提交。

**所以在 MySQL 中，你一定要对大事务特别对待，** 总结起来就是：

1. 设计时，把 DELETE 删除操作转化为 DROP TABLE/PARTITION 操作；
2. 业务设计时，把大事务拆成小事务。

对于第一点（把 DELETE 删除操作转化为 DROP TABLE/PARTITION 操作），主要是在设计时把流水或日志类的表按时间分表或者分区，这样在删除时，二进制日志内容就是一条 DROP TABLE/PARITION 的 SQL，写入速度就非常快了。

而第二点（把大事务拆分成小事务）也能控制二进制日志的大小。比如对于前面的 DELETE 操作，如果设计时没有分表或分区，那么你可以进行如下面的小事务拆分：

```
DELETE FROM ...
WHEREE time between ... and ...
LIMIT 1000;
```

**MySQL 数据库中，大事务除了会导致提交速度变慢，还会导致主从复制延迟**。

试想一下，一个大事务在主服务器上运行了 30 分钟，那么在从服务器上也需要运行 30 分钟。在从机回放这个大事务的过程中，主从服务器之间的数据就产生了延迟；产生大事务的另一种可能性是主服务上没有创建索引，导致一个简单的操作时间变得非常长。这样在从机回放时，也会需要很长的时间从而导致主从的复制延迟。

### 主从复制延迟优化



MySQL 的从机并行复制有两种模式。

1. **COMMIT ORDER：** 主机怎么并行，从机就怎么并行。
2. **WRITESET：** 基于每个事务，只要事务更新的记录不冲突，就可以并行。

如果插入的记录没有冲突，比如唯一索引冲突，**那么虽然主机是单线程，但从机可以是多线程并行回放！！！**

所以在 WRITESET 模式下，主从复制几乎没有延迟。那么要启用 WRITESET 复制模式，你需要做这样的配置：

复制代码

```
binlog_transaction_dependency_tracking = WRITESET

transaction_write_set_extraction = XXHASH64

slave-parallel-type = LOGICAL_CLOCK

slave-parallel-workers = 16
```

因为主从复制延迟会影响到后续高可用的切换，以及读写分离的架构设计，所以在真实的业务中，你要对主从复制延迟进行监控。

### 主从复制延迟监控

SHOW SLAVE STATUS，其中的 Seconds_Behind_Master

心跳表  在主服务器上引入一张心跳表 heartbeat，用于定期更新时间

主从延迟 = 从机当前时间 - 表 heartbeat 中的时间

```
USE DBA;
CREATE TABLE heartbeat (
  server-uuid VARCHAR(36) PRIMARY KEY,
  ts TIMESTAMP(6) NOT NULL
);
REPLACE INTO heartbeat(@@server_uuid, NOW())
```



### 读写分离设计

读写分离设计的前提是从机不能落后主机很多，最好是能准实时数据同步，务必一定要开始并行复制，并确保线上已经将大事务拆成小事务。

读写分离设计的兜底

在 Load Balance 服务器，可以配置较小比例的读取请求访问主机，如上图所示的 1%，其余三台从服务器各自承担 33% 的读取请求。

如果发生严重的主从复制情况，可以设置下面从机权重为 0，将主机权重设置为 100%，这样就不会因为数据延迟，导致对于业务的影响了。







