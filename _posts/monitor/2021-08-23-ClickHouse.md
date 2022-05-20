

## 系统要求

官方预构建的二进制文件通常针对x86_64进行编译，并利用`SSE 4.2`指令集。若不支持`SSE 4.2`则可以通过源码进行构建，同时禁用`SSE 4.2`或`AArch64 cpu`。

```shell
# 支持SSE4.2,可以直接使用官方与构建的二进制文件
[root@dpm03 ~]#  grep -q sse4_2 /proc/cpuinfo && echo "SSE 4.2 supported" || echo "SSE 4.2 not supported"
SSE 4.2 supported
```

## 安装

### 搭建一个zookeeper集群

```shell
# 下载压缩包
$ curl -O https://mirrors.bfsu.edu.cn/apache/zookeeper/zookeeper-3.5.9/apache-zookeeper-3.5.9-bin.tar.gz

# 解压
$ tar -zxvf apache-zookeeper-3.5.9-bin.tar.gz

# 进入conf目录，修改配置
$ cd apache-zookeeper-3.5.9-bin/conf

$ cp zoo_sample.cfg zoo.cfg

$ vim zoo.cfg 

# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/data/clickhouse/apache-zookeeper-3.5.9-bin/data/zookeeper
dataLogDir=/data/clickhouse/apache-zookeeper-3.5.9-bin/log/zookeeper
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
auitopurge.purgeInterval=0
globalOutstandingLimit=200
server.1=dpm01:2888:3888
server.2=dpm02:2888:3888
server.3=dpm03:2888:3888


# 创建数据目录
$ mkdir -p /data/clickhouse/apache-zookeeper-3.5.9-bin/data/zookeeper
$ mkdir -p /data/clickhouse/apache-zookeeper-3.5.9-bin/log/zookeeper

# 设置myid
$ vim /data/clickhouse/apache-zookeeper-3.5.9-bin/data/zookeepe/myid
dpm01 1   dpm02 2 dpm03 3

# 进入zookeeper的bin目录，启动zookeeper服务，每个节点都需要启动
$ ./zkServer.sh start

# 查看状态
$ ./zkServer status
```

### `yum`安装

```shell
$ sudo yum install clickhouse-server clickhouse-client
```

### `Tgz`安装

### 单机启动

```shell
# 使用clickhouse用户启动， --config-file 指定配置文件 & 后台运行
$ sudo -u clickhouse clickhouse-server --config-file=/etc/clickhouse-server/config.xml &


ps -ef |grep clickhouse


```

### 集群部署

1. 修改配置`config.xml`

   > 1. 预增加分片及副本配置，下面是3分片1副本的配置。<font color="red">每个副本不能出现在同一台机器上，若3分片3副本的集群，则需要9个节点</font>
   >
   > 2. 增加`Zookeeper`集群配置,`ClickHouse`的集群依赖于`Zookeeper`
   >
   > 3. 增加变量配置
   >
   >    <font color="red">注意：变量配置每个几点不同，其他配置信息每个节点都需要相同</font>

```xml
	<!-- 预增加分片及副本配置，下面是3分片1副本的配置，每个副本 -->
	<remote_servers>
        <log>
            <shard>
                <replica>
                    <host>dpm01</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                 <replica>
                    <host>dpm02</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>dpm03</host>
                    <port>9000</port>
                </replica>

            </shard>
        </log>
    </remote_servers>

    <!--修改Zookeeper配置-->
	 <zookeeper>
        <node>
            <host>dpm01</host>
            <port>2181</port>
        </node>
        <node>
            <host>dpm02</host>
            <port>2181</port>
        </node>
        <node>
            <host>dpm03</host>
            <port>2181</port>
        </node>
    </zookeeper>

	<!-- 增加变量（此处，集群每个节点不同 -->
	<macros>
        <shard>01</shard>
        <replica>dpm01</replica>
    </macros>
```

### 集群表

> 1. 创建本地表，保存实际数据；
> 2. 创建分布式表，关联本地表，读取或者写入数据；
> 3. **分布式引擎本身不存储数据**, 但可以在多个服务器上进行分布式查询。读是自动并行的。读取时，远程服务器表的索引（如果有的话）会被使用

```sql
# 创建本地表 ON CLUSTER log表示在集群中每个节点都创建此表，集群名称为：log，需要预设置
CREATE TABLE demo_local ON CLUSTER log                                            \
(                                                                                 \
    EventDate DateTime,                                                           \
    CounterID UInt32,                                                             \
    UserID UInt32                                                                 \
)ENGINE = MergeTree()    					  									  \
PARTITION BY toYYYYMM(EventDate)                                                  \
ORDER BY (CounterID, EventDate, intHash32(UserID))                                \
SAMPLE BY intHash32(UserID);

# 创建分布式表，创建分布式表时，也可以使用`ON CLUSTER log`参数，在整个集群中都进行创建
CREATE TABLE demo_all AS demo_local ON CLUSTER log ENGINE = Distributed(log, default, demo_local,rand());
```



## 使用

```shell
# 打开客户端
[root@dpm02 ~]#  clickhouse-client
ClickHouse client version 21.6.3.14 (official build).
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 21.6.3 revision 54448.

dpm02 :) 

```

### 基础命令

#### clickhouse-client --query

> 执行SQL语句
>
> ```shell
> # 创建
> $ clickhouse-client --query "CREATE DATABASE IF NOT EXISTS tutorial"
> ```
>
> 

### 性能

#### 批量插入

> 1. 每次批量插入2W条数据平均耗时在1秒钟

### 引擎

#### GraphiteMergeTree引擎

> 该引擎用来对 [Graphite](https://graphite.readthedocs.io/en/latest/index.html)数据进行瘦身及汇总。对于想使用CH来存储Graphite数据的开发者来说可能有用。
>
> 如果不需要对Graphite数据做汇总，那么可以使用任意的CH表引擎；但若需要，那就采用 `GraphiteMergeTree` 引擎。它能减少存储空间，同时能提高Graphite数据的查询效率。

## ClickHouse的特性

### 真正的列式数据库管理系统

> 在一个真正的列式数据库管理系统中，除了数据本身外不应该存在其他额外的数据。这意味着为了避免在值旁边存储它们的长度«number»，你必须支持固定长度数值类型。

### 数据压缩

> ClickHouse还提供针对特定类型数据的[专用编解码器](https://clickhouse.tech/docs/zh/sql-reference/statements/create/#create-query-specialized-codecs)

### 数据的磁盘存储

> ClickHouse被设计用于工作在传统磁盘上的系统，它提供每GB更低的存储成本，但如果可以使用SSD和内存，它也会合理的利用这些资源。

### 多核心并行处理

> ClickHouse会使用服务器上一切可用的资源，从而以最自然的方式并行处理大型查询。

### 多服务器分布式处理

> 在ClickHouse中，数据可以保存在不同的shard上，每一个shard都由一组用于容错的replica组成，查询可以并行地在所有shard上进行处理。这些对用户来说是透明的

### 支持SQL

> ClickHouse支持一种[基于SQL的声明式查询语言](https://clickhouse.tech/docs/zh/sql-reference/)，它在许多情况下与[ANSI SQL标准](https://clickhouse.tech/docs/zh/sql-reference/ansi/)相同。
>
> 支持的查询[GROUP BY](https://clickhouse.tech/docs/zh/sql-reference/statements/select/group-by/), [ORDER BY](https://clickhouse.tech/docs/zh/sql-reference/statements/select/order-by/), [FROM](https://clickhouse.tech/docs/zh/sql-reference/statements/select/from/), [JOIN](https://clickhouse.tech/docs/zh/sql-reference/statements/select/join/), [IN](https://clickhouse.tech/docs/zh/sql-reference/operators/in/)以及非相关子查询。
>
> 相关(依赖性)子查询和窗口函数暂不受支持，但将来会被实现。

### 向量引擎

> 为了高效的使用CPU，数据不仅仅按列存储，同时还按向量(列的一部分)进行处理，这样可以更加高效地使用CPU。

### 实时的数据更新

>ClickHouse支持在表中定义主键。为了使查询能够快速在主键中进行范围查找，数据总是以增量的方式有序的存储在MergeTree中。因此，数据可以持续不断地高效的写入到表中，并且写入的过程中不会存在任何加锁的行为。

### 索引

> 按照主键对数据进行排序，这将帮助ClickHouse在几十毫秒以内完成对数据特定值或范围的查找。

### 适合在线查询

> 在线查询意味着在没有对数据做任何预处理的情况下以极低的延迟处理查询并将结果加载到用户的页面中。

### 支持近似计算

> ClickHouse提供各种各样在允许牺牲数据精度的情况下对查询进行加速的方法：
>
> 1. 用于近似计算的各类聚合函数，如：distinct values, medians, quantiles
> 2. 基于数据的部分样本进行近似查询。这时，仅会从磁盘检索少部分比例的数据。
> 3. 不使用全部的聚合条件，通过随机选择有限个数据聚合条件进行聚合。这在数据聚合条件满足某些分布条件下，在提供相当准确的聚合结果的同时降低了计算资源的使用。

### 支持数据复制和数据完整性

>ClickHouse使用异步的多主复制技术。当数据被写入任何一个可用副本后，系统会在后台将数据分发给其他副本，以保证系统在不同副本上保持相同的数据。在大多数情况下ClickHouse能在故障后自动恢复，在一些少数的复杂情况下需要手动恢复。

### 角色的访问控制

> ClickHouse使用SQL查询实现用户帐户管理，并允许[角色的访问控制](https://clickhouse.tech/docs/zh/operations/access-rights/)，类似于ANSI SQL标准和流行的关系数据库管理系统。

## 优缺点

优点：

1. 为了高效的使用CPU，数据不仅仅按列存储，同时还按向量进行处理；
2. 数据压缩空间大，减少IO；处理单查询高吞吐量每台服务器每秒最多数十亿行；
3. 索引非B树结构，不需要满足最左原则；只要过滤条件在索引列中包含即可；即使在使用的数据不在索引中，由于各种并行处理机制ClickHouse全表扫描的速度也很快；
4. 写入速度非常快，50-200M/s，对于大量的数据更新非常适用。

缺点：

1. 不支持事务，不支持真正的删除/更新；
2. 不支持高并发，官方建议qps为100，可以通过修改配置文件增加连接数，但是在服务器足够好的情况下；
3. SQL满足日常使用80%以上的语法，join写法比较特殊；最新版已支持类似SQL的join，但性能不好；
4. 尽量做1000条以上批量的写入，避免逐行insert或小批量的insert，update，delete操作，因为ClickHouse底层会不断的做异步的数据合并，会影响查询性能，这个在做实时数据写入的时候要尽量避开；
5. Clickhouse快是因为采用了并行处理机制，即使一个查询，也会用服务器一半的CPU去执行，所以ClickHouse不能支持高并发的使用场景，默认单查询使用CPU核数为服务器核数的一半，安装时会自动识别服务器核数，可以通过配置文件修改该参数。

全量数据导入：数据导入临时表 -> 导入完成后，将原表改名为tmp1 -> 将临时表改名为正式表 -> 删除原表

增量数据导入： 增量数据导入临时表 -> 将原数据除增量外的也导入临时表 -> 导入完成后，将原表改名为tmp1-> 将临时表改成正式表-> 删除原数据表

相关优化：

1. 关闭虚拟内存，物理内存和虚拟内存的数据交换，会导致查询变慢。
2. 为每一个账户添加join_use_nulls配置，左表中的一条记录在右表中不存在，右表的相应字段会返回该字段相应数据类型的默认值，而不是标准SQL中的Null值。
3. JOIN操作时一定要把数据量小的表放在右边，ClickHouse中无论是Left Join 、Right Join还是Inner Join永远都是拿着右表中的每一条记录到左表中查找该记录是否存在，所以右表必须是小表。
4. 批量写入数据时，必须控制每个批次的数据中涉及到的分区的数量，在写入之前最好对需要导入的数据进行排序。无序的数据或者涉及的分区太多，会导致ClickHouse无法及时对新导入的数据进行合并，从而影响查询性能。
5. 尽量减少JOIN时的左右表的数据量，必要时可以提前对某张表进行聚合操作，减少数据条数。有些时候，先GROUP BY再JOIN比先JOIN再GROUP BY查询时间更短。
6. ClickHouse的分布式表性能性价比不如物理表高，建表分区字段值不宜过多，防止数据导入过程磁盘可能会被打满。
7. CPU一般在50%左右会出现查询波动，达到70%会出现大范围的查询超时，CPU是最关键的指标，要非常关注。

性能情况

1. 单个查询吞吐量：如果数据被放置在page cache中，则一个不太复杂的查询在单个服务器上大约能够以2-10GB／s（未压缩）的速度进行处理（对于简单的查询，速度可以达到30GB／s）。如果数据没有在page cache中的话，那么速度将取决于你的磁盘系统和数据的压缩率。例如，如果一个磁盘允许以400MB／s的速度读取数据，并且数据压缩率是3，则数据的处理速度为1.2GB/s。这意味着，如果你是在提取一个10字节的列，那么它的处理速度大约是1-2亿行每秒。对于分布式处理，处理速度几乎是线性扩展的，但这受限于聚合或排序的结果不是那么大的情况下。
2. 处理短查询的延时时间：数据被page cache缓存的情况下，它的延迟应该小于50毫秒(最佳情况下应该小于10毫秒)。 否则，延迟取决于数据的查找次数。延迟可以通过以下公式计算得知： 查找时间（10 ms） * 查询的列的数量 * 查询的数据块的数量。
3. 处理大量短查询：ClickHouse可以在单个服务器上每秒处理数百个查询（在最佳的情况下最多可以处理数千个）。但是由于这不适用于分析型场景。建议每秒最多查询100次。
4. 数据写入性能：建议每次写入不少于1000行的批量写入，或每秒不超过一个写入请求。当使用tab-separated格式将一份数据写入到MergeTree表中时，写入速度大约为50到200MB/s。如果您写入的数据每行为1Kb，那么写入的速度为50，000到200，000行每秒。如果您的行更小，那么写入速度将更高。为了提高写入性能，您可以使用多个INSERT进行并行写入，这将带来线性的性能提升。

count: 千万级别，500毫秒，1亿 800毫秒  2亿 900毫秒 3亿 1.1秒
group: 百万级别 200毫米，千万 1秒，1亿 10秒，2亿 20秒，3亿 30秒
join：千万-10万 600 毫秒， 千万 -百万：10秒，千万-千万 150秒

ClickHouse并非无所不能，查询语句需要不断的调优，可能与查询条件有关，不同的查询条件表是左join还是右join也是很有讲究的。

其他补充：

1. MySQL单条SQL是单线程的，只能跑满一个core，ClickHouse相反，有多少CPU，吃多少资源，所以飞快；

2. ClickHouse不支持事务，不存在隔离级别。ClickHouse的定位是分析性数据库，而不是严格的关系型数据库。

3. IO方面，MySQL是行存储，ClickHouse是列存储，后者在count()这类操作天然有优势，同时，在IO方面，MySQL需要大量随机IO，ClickHouse基本是顺序IO。

   有人可能觉得上面的数据导入的时候，数据肯定缓存在内存里了，这个的确，但是ClickHouse基本上是顺序IO。对IO基本没有太高要求，当然，磁盘越快，上层处理越快，但是99%的情况是，CPU先跑满了（数据库里太少见了，大多数都是IO不够用）。

## 参考文档

[ClickHouse中文文档]: https://clickhouse.tech/docs/zh

