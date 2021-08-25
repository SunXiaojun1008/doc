### 什么是InfluxDB

```
InfluxDB是一个由InfluxData开发的开源时序型数据。它由Go写成，着力于高性能地查询与存储时序型数据。InfluxDB被广泛应用于存储系统的监控数据，IoT行业的实时数据等场景。
```

### 对常见关系型数据库（MySQL）的基础概念对比

| 概念         | MySQL    | InfluxDB                                                    |
| ------------ | -------- | ----------------------------------------------------------- |
| 数据库（同） | database | database                                                    |
| 表（不同）   | table    | measurement                                                 |
| 列（不同）   | column   | tag(带索引的，非必须)、field(不带索引)、timestemp(唯一主键) |

- tag set：**不同**的每组tag key和tag value的集合；
- field set：每组field key和field value的集合；
- retention policy：数据存储策略（默认策略为autogen）InfluxDB没有删除数据操作，规定数据的保留时间达到清除数据的目的；
- series：共同retention policy，measurement和tag set的集合；
- 示例数据如下： 其中census是measurement，butterflies和honeybees是field key，location和scientist是tag key

### 注意点

- tag 只能为字符串类型

- field 类型无限制

- 不支持join

- 支持连续查询操作（汇总统计数据）：CONTINUOUS QUERY

- 配合Telegraf服务（Telegraf可以监控系统CPU、内存、网络等数据）

- 配合Grafana服务（数据展现的图像界面，将influxdb中的数据可视化）

### 安装

1. 下载&安装  https://portal.influxdata.com/downloads/

   ```shell
   # CentOS
   wget https://dl.influxdata.com/influxdb/releases/influxdb-1.8.3.x86_64.rpm
   
   sudo yum localinstall influxdb-1.8.3.x86_64.rpm
   ```

2. 配置
   1. 设置influxdb启动时使用的配置

      ```shell
      # 启动时指定配置文件
      influxd -config /etc/influxdb/influxdb.conf
      
      # 将配置文件设置为环境变量
      echo $INFLUXDB_CONFIG_PATH
      /etc/influxdb/influxdb.conf
      
      influxd
      ```

   2. 修改配置文件

      ```shell
      vim /etc/influxdb/influxdb.conf
      
      ...
      
      [meta]
        dir = "/mnt/db/meta"
        ...
      
      ...
      
      [data]
        dir = "/mnt/db/data"
        ...
      wal-dir = "/mnt/influx/wal"
        ...
      
      ...
      
      [hinted-handoff]
          ...
      dir = "/mnt/db/hh"
          ...
      ```

      

   3. 授权

      当使用非标准目录用于InfluxDB数据和配置时，需要保证所在文件系统拥有足够的权限。

      ```shell
      chown influxdb:influxdb /mnt/influx
      chown influxdb:influxdb /mnt/db
      ```

      对于InfluxDB 1.7.6或更高版本，您必须授予所有者对init.sh文件的权限。需要到influxDB目录执行下面的脚本:**/usr/lib/influxdb/scripts**

      ```shell
      if [ ! -f "$STDOUT" ]; then
          mkdir -p $(dirname $STDOUT)
          chown $USER:$GROUP $(dirname $STDOUT)
       fi
      
       if [ ! -f "$STDERR" ]; then
          mkdir -p $(dirname $STDERR)
          chown $USER:$GROUP $(dirname $STDERR)
       fi
      
       # Override init script variables with DEFAULT values
      ```

### 常用InfluxQL

```sql
-- 查看所有的数据库
show databases;
-- 使用特定的数据库
use database_name;
-- 查看所有的measurement
show measurements;
-- 查询10条数据
select * from measurement_name limit 10;
-- 数据中的时间字段默认显示的是一个纳秒时间戳，改成可读格式
precision rfc3339; -- 之后再查询，时间就是rfc3339标准格式
-- 或可以在连接数据库的时候，直接带该参数
us
-- 查看一个measurement中所有的tag key 
show tag keys
-- 查看一个measurement中所有的field key 
show field keys
-- 查看一个measurement中所有的保存策略(可以有多个，一个标识为default)
show retention policies;
```

