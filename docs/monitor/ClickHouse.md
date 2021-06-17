

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

## 参考文档

[ClickHouse中文文档]: https://clickhouse.tech/docs/zh	"ClickHouse中文文档"

