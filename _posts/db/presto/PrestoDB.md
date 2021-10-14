### 下载

> 从官网上下载Presto安装包，也可以从[Github下载](https://github.com/prestodb/presto)后自行编译

```shell
$ wget https://codeload.github.com/zeeshanabid94/presto/zip/refs/heads/presto-clickhouse
```

### 安装

- **<font color="red">Presto requires Java 8u151+</font>**

- **<font color="red">Presto requires a 64-bit JVM </font>**

> Presto 需要一个**数据目录**来存储日志等，在安装目录之外创建一个数据目录，这样升级 Presto 时可以轻松保存。

```shell
[root@dpm02 ~]#  cd /root/presto


[root@dpm02 ~/presto]# tar zxvf presto-server-0.260.tar.gz
```

### 配置

在安装目录中创建一个目录，包含一下配置：

- Node Properties（节点属性）：每个节点特有的环境配置
- JVM Config（JVM配置）：Java 虚拟机的命令行选项配置
- Config Properties（配置信息）：Presto服务端的配置信息
- Log Properties：日志相关配置
- Catalog Properties：连接器（数据源）的配置信息

#### Node Properties

节点属性文件`etc/node.properties`包含特定于每个节点的配置。节点是一台机器上的Presto的单个安装的实例。此文件通常由部署系统在首次安装 Presto 时创建。以下是最小的`etc/node.properties`

```properties
node.environment=production
node.id=ffffffff-ffff-ffff-ffff-ffffffffffff
node.data-dir=/root/presto/data
```

- `node.environment`：节点环境名称。集群中的所有 Presto 节点必须具有相同的环境名称
- `node.id`：Presto节点唯一标识符。这对于每个节点必须是唯一的。此标识符应在 Presto 重新启动或升级期间保持一致。如果在一台机器上运行多个 Presto 安装（即同一台机器上的多个节点），每个安装必须有一个唯一的标识符。
- `node.data-dir`：数据目录的位置（文件系统路径）。Presto 将在这里存储日志和其他数据。

#### JVM Config

JVM 配置文件`etc/jvm.config`包含用于启动 Java 虚拟机的命令行选项列表。该文件的格式是一个选项列表，每行一个。这些选项不被 shell 解释，不能包含空格或其他特殊字符的。

```
-server
-Xmx16G
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-XX:+HeapDumpOnOutOfMemoryError
-XX:+ExitOnOutOfMemoryError
```

> 因为`OutOfMemoryError`通常会使 JVM 处于不一致状态，所以我们编写了一个堆转储（用于调试）并在发生这种情况时强行终止进程。

#### Config Properties

配置属性文件`etc/config.properties`包含 Presto 服务器的配置。每个 Presto 服务器既可以充当协调节点，也可以充当工作节点，但是将一个节点专门用于执行协调工作，可以在更大的集群上提供最佳性能。

- coordinator 节点配置如下：

  ```properties
  coordinator=true
  node-scheduler.include-coordinator=false
  http-server.http.port=8080
  query.max-memory=50GB
  query.max-memory-per-node=1GB
  query.max-total-memory-per-node=2GB
  discovery-server.enabled=true
  discovery.uri=http://example.net:8080
  ```

- worker 节点配置如下：

  ```properties
  coordinator=false
  http-server.http.port=8080
  query.max-memory=50GB
  query.max-memory-per-node=1GB
  query.max-total-memory-per-node=2GB
  discovery.uri=http://example.net:8080
  ```

-  一个节点同时作为coordinator和worker 的配置如下：

  ```properties
  coordinator=true
  node-scheduler.include-coordinator=true
  http-server.http.port=8080
  query.max-memory=5GB
  query.max-memory-per-node=1GB
  query.max-total-memory-per-node=2GB
  discovery-server.enabled=true
  discovery.uri=http://example.net:8080
  ```

属性说明

- `coordinator`： 是否将当前节点作为协调节点。
- `node-scheduler.include-coordinator`：允许在协调器上安排工作。对于较大的集群，协调器上的处理工作会影响查询性能，因为机器的资源无法用于调度、管理和监控查询执行的关键任务。
- `http-server.http.port`: 指定 HTTP 服务器的端口。Presto 使用 HTTP 进行所有内部和外部通信。
- `query.max-memory`：查询可能使用的最大分布式内存量。
- `query.max-memory-per-node`：查询可以在任何一台机器上使用的最大用户内存量。
- `query.max-total-memory-per-node`：查询在任何一台机器上可能使用的最大用户和系统内存量，其中系统内存是读取器、写入器和网络缓冲区等在执行过程中使用的内存。
- `discovery-server.enabled`：Presto 使用 Discovery 服务查找集群中的所有节点。每个 Presto 实例都会在启动时向 Discovery 服务注册自己。为了简化部署并避免运行额外的服务，Presto 协调器可以运行一个嵌入式版本的 Discovery 服务。它与 Presto 共享 HTTP 服务器，因此使用相同的端口。
- `discovery.uri`：发现服务器的 URI。因为我们在 Presto 协调器中启用了 Discovery 的嵌入式版本，所以这应该是 Presto 协调器的 URI。替换`example.net:8080`以匹配 Presto 协调器的主机和端口。<font color="red">此 URI 不得以斜杠结尾</font>。
- `jmx.rmiregistry.port`：指定 JMX RMI 注册表的端口。JMX 客户端应该连接到这个端口。
- `jmx.rmiserver.port`：指定 JMX RMI 服务器的端口。Presto 导出许多可用于通过 JMX 进行监控的指标。

除此以外，还有很多其他配置，可以参考[配置组](https://prestodb.io/docs/current/admin/resource-groups.html)

#### Log Properties

可选的日志文件`etc/log.properties`允许为命名记录器层次结构设置最低日志级别。基于Java的包路径进行日志级别的配置。例如，考虑以下日志级别文件：

```properties
com.facebook.presto=INFO
```

#### Catalog Properties

Presto 通过*连接器*访问数据，*连接器*安装在目录中。连接器提供目录中的所有模式和表。详细内容可以参看官网[连接器](https://prestodb.io/docs/current/connector.html)

### 启动Presto

以守护进程运维Presto

```shell
[root@dpm02 ~]#  cd /root/presto/presto-server-0.260/bin

[root@dpm02 ~/presto/presto-server-0.260/bin]#  ./launcher start

```

也可以让Presto在前台运行，将日志等信息输出到控制台

```shell
[root@dpm02 ~/presto/presto-server-0.260/bin]#  ./launcher run
```

运行启动器`--help`以查看支持的命令和命令行选项。特别是，该`--verbose`选项对于调试安装非常有用。

启动后，您可以在`var/log`以下位置找到日志文件：

- `launcher.log`: 这个日志是由启动器创建的，并连接到服务器的 stdout 和 stderr 流。它将包含一些在初始化服务器日志记录以及 JVM 产生的任何错误或诊断时发生的日志消息。
- `server.log`：这是 Presto 使用的主要日志文件。如果服务器在初始化期间出现故障，它通常会包含相关信息。它会自动旋转和压缩。
- `http-request.log`：这是包含服务器收到的每个 HTTP 请求的 HTTP 请求日志。它会自动旋转和压缩。

### Presto命令行

1. 连接

   ```shell
   ./presto --server localhost:18090 --catalog postgresql
   ```

   

### 异常

### Presto：特性、原理、架构

#### 应用场景

主要还是查询时延要求在毫秒或者秒级别的SQL并发BI报表，Ad-Hoc（点对点）查询以及多数据源关联查询居多。

#### 特性

- 多数据源
- ANSI SQL（Presto可以完全支持ANSI SQL）
- 混合计算（混个多个catalog进行join查询和计算）
- 流水线式作业（基于Pipeline进行设计，在进行海量数据处理的过程中，可以把部分结果数据返回给用户）
- 低延时（完全基于内存的并行计算、流水线式计算作业、本地化计算、动态编译执行计划、GC控制）
- 支持数据溢出（将溢出的数据写入到磁盘）

#### 架构

![Presto架构](..\..\images\presto\Presto架构.png)

##### Coordinator服务

- Presto集群的Master节点
- 接收查询请求、解析查询语句
- 生成查询执行计划
- 任务调度（调度Worker节点工作）
- 管理Worker节点

##### Worker服务

- 工作节点，执行被Coordinator分解后的任务（Task）
- 心跳，向Coordinator注册自己
- 对Task读入的每个Spit进行一系列的操作和处理

#### Presto模型

presto可通过connector连接一个数据源，可以根据一个connector配置多个catalog，一个catalog可以有多个schema，一个schema可以有多个table。

```ASN.1
|___connector
	|___catalog
		|___ schema
			|— table1
			|_ table2
			|_ ...
		|___ schema
			|— table1
			|_ table2
			|_ ...
	|___catalog
		|___ schema
			|— table1
			|_ table2
			|_ ...
		|___ schema
			|— table1
			|_ table2
			|_ ...
```



##### Connector

可以将Connector当作Presto访问各种不同数据源的驱动程序，Presto是通过多种多样的Connector来访问多种不同的数据源的。每种Connector都实现了Presto中标准的SPI接口，因此只要你实现Presto中标准的SPI接口，就可以轻易地实现使用适合自己特定需求的Connector来访问特定的数据源。

> Presto目前内置可以支持的Connector有Hive、Kafka、MySQL、Cassandra、Elasticsearch、Postgres等

##### Catalog

Presto中的Catalog类似于MySQL中的一个数据库实例。

##### Schema

Presto中的Schema类似于MySQL中的Database，一个Catalog中可以包含多个Schema。

##### Table

Presto中的Table与传统数据库中的Table的含义是一样的。

#### presto查询执行模型

- Statement语句 其实就是输入的SQL
- Query 根据SQL语句生成查询执行计划，进而生成可以执行的查询（Query），一个查询执行由Stage、Task、Driver、Split、Operator和DataSource组成
- Stage 执行查询阶段 Stage之间是树状的结构 ，RootStage 将结果返回给coordinator ，SourceStage接收coordinator数据 其他stage都有上下游 stage分为四种 single(root)、Fixed、source、coordinator_only（DML or DDL）
- Exchange 两个stage数据的交换通过Exchange 两种Exchange ；Output Buffer （生产数据的stage通过此传给下游stage）Exchange Client （下游消费）；如果stage 是source 直接通过connector 读数据，则改stage通过Operator与connector交互
- stage 并不会被执行，只是对执行计划进行管理
- Task 实际运行在worker上的
- Driver 一个Driver处理一个split
- Operator 一个operator代表对一个split的一种操作 operator每次只会读取一个paged对象
- Split 分片一个分片就是一个大的数据集中的一个小的子集
- Page presto中处理的最小数据单元 一个page包含多个block对象，每个block对象是个字节数据
