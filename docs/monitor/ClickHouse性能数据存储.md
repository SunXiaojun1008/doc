### ClickHouse部署

> 3主3从方式部署：3个分片主节点，每个主节点各有一个副本节点，共需6台节点进行部署。
>
> 组件：
>
> 1. Zookeeper集群（3个节点）
> 2. ClickHouse集群（6个节点）：ClickHouse的副本是表级别的，需要保证同一张表的多个副本不在同一台几点上班，否则会表会创建失败。若不使用单分片，则下图只需一个Shard节点即可。
> 3. Nginx（尽量部署高可用）：写入代理，使用Nginx进行负载，写数据到各个分片的主副本。不使用ClickHouse的分布式表自带负载。直接写ClickHouse本地表的性能比通过分布式表再写本地表性能要高；查询时，使用分布式表进行查询，分布式表会对数据进行聚合。

![ClickHouse集群](..\images\ClickHouse集群.png)

```xml
	<!-- 集群配置，需要预设置 -->
	<remote_servers>
        <3shard2replica>  <!-- 集群名称：可自定义 -->
            <shard> <!-- 分片设置 -->
                <replica>  <!-- 副本设置 -->
                    <host>node01</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>node04</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>node02</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>node05</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>node03</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>node06</host>
                    <port>9000</port>
                </replica>
            </shard>
        </3shard2replica>
    </remote_servers>

	<!-- Zookeeper配置 -->
	<zookeeper>
        <node>
            <host>node01</host>
            <port>2181</port>
        </node>
        <node>
            <host>node02</host>
            <port>2181</port>
        </node>
        <node>
            <host>node03</host>
            <port>2181</port>
        </node>
    </zookeeper>

	<!-- 其他配置：每个ClickHouse节点，此配置都不同 -->
	<macros>
        <shard>01</shard>
        <replica>node01</replica>
    </macros>
```



### 性能数据接入

> 1. 预采集，生成Schema：下发采集任务，采集性能指标信息，解析后生成ClickHouse的表schema信息。
> 2. 组装SQL语句，进行建表。本地表：写入使用；分布式表：查询使用。

#### 性能数据结构

```json
{
    "target":"指标名",
    "value":"指标值",
    "timestamp":"时间戳",
    "type":"指标类型",
    "label1":"标签1",
    ...
}
```

#### ClickHouse表引擎

> 使用<font color="red">`GraphiteMergeTree`</font>作为本地表的表引擎
>
> 1. MergeTree系列的引擎被设计用于插入极大量的数据到一张表当中。数据可以以数据片段的形式一个接着一个的快速写入，数据片段在后台按照一定的规则进行合并。相比在插入时不断修改（重写）已存储的数据，这种策略会高效很多。
>    - 存储的数据按主键排序
>    - 支持数据分区
>    - 支持数据副本
>    - 支持数据采样
>
> 2. 冷热数据分离存储
>
> 3. 根据配置的规则对数据进行聚合汇总：CH在处理行记录时，会检查 `pattern`节点的规则。每个 `pattern`（含`default`）节点可以包含 `function` 用于聚合操作，或`retention`参数，或者两者都有。如果指标名称和 `regexp`相匹配，相应 `pattern`的规则会生效；否则，使用 `default` 节点的规则。
>
>    ```xml
>    <!-- 案例 -->
>    <graphite_rollup>
>        <version_column_name>Version</version_column_name>
>        <pattern>
>            <regexp>click_cost</regexp>
>            <function>any</function>
>            <retention>
>                <age>0</age>
>                <precision>5</precision>
>            </retention>
>            <retention>
>                <age>86400</age>
>                <precision>60</precision>
>            </retention>
>        </pattern>
>        <default>
>            <function>max</function>
>            <retention>
>                <age>0</age>
>                <precision>60</precision>
>            </retention>
>            <retention>
>                <age>3600</age>
>                <precision>300</precision>
>            </retention>
>            <retention>
>                <age>86400</age>
>                <precision>3600</precision>
>            </retention>
>        </default>
>    </graphite_rollup>
>    
>    ```
>
> 

#### 建表语句模板

```sql
-- 本地表，使用GraphiteMergeTree引擎
CREATE TABLE metrics_demo_local ON CLUSTER cluster 
(
	target  String,
    time 	DateTime,
    value	Float64,
    version	UInt64,
    label1	String,
    ...
)
ENGINE = GraphiteMergeTree(config_section) -- config_section: 配置文件中标识汇总规则的节点名称 ，汇总规则需要预先配置在配置文件中
[PARTITION BY expr] -- 分区键，时序数据可以考虑根据时间做分区
[ORDER BY expr]	-- 排序键， 根据时间做拍戏
[SAMPLE BY expr] -- 抽样
[SETTINGS name=value, ...] -- 设置


-- 副本表
CREATE TABLE metrics_demo_local ON CLUSTER cluster 
(
	target  String,
    time 	DateTime,
    value	Float64,
    version	UInt64,
    label1	String,
    ...
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/metrics_demo_local', '{replica}') -- replica: 预先配置在配置文件中
[PARTITION BY expr] -- 分区键，时序数据可以考虑根据时间做分区
[ORDER BY expr]	-- 排序键， 根据时间做拍戏
[SAMPLE BY expr] -- 抽样
[SETTINGS name=value, ...] -- 设置


-- 分布式表
CREATE TABLE metrics_demo_all AS metrics_demo_local ON CLUSTER cluster
ENGINE = Distributed(cluster, default, metrics_demo_local,rand());
-- 参数 cluster：集群名称
-- 参数 default： 数据库名称
-- 参数 metrics_demo_local： 本地表名称
-- 参数 rand() ： 数据分片规则
```

### 注意事项

1. ClickHouse不支持高并发，官方建议qps为100
2. 尽量做1000条以上批量的写入，避免逐行insert或小批量的insert操作，因为ClickHouse底层会不断的做异步的数据合并，会影响查询性能，这个在做实时数据写入的时候要尽量避开,做到<font color="red">多批量，少批次</font>。

### 优化

1. 关闭虚拟内存，物理内存和虚拟内存的数据交换，会导致查询变慢。
2. ClickHouse的分布式表性能性价比不如物理表高，建表分区字段值不宜过多，防止数据导入过程磁盘可能会被打满。
3. 批量写入数据时，必须控制每个批次的数据中涉及到的分区的数量，在写入之前最好对需要导入的数据进行排序。无序的数据或者涉及的分区太多，会导致ClickHouse无法及时对新导入的数据进行合并，从而影响查询性能。

### 存在问题

1. ClickHouse 不支持高并发，针对上云产品存储时，若每种上云产品建一张表，随着上云产品接入数量递增会存在写入性能瓶颈。

