### 环境安装

1. 下载安装包

```shell
wget https://github.com/prometheus/prometheus/releases/download/v2.22.0/prometheus-2.22.0.linux-amd64.tar.gz
```

2. 解压文件

```shell
tar xvfz prometheus-2.22.0.linux-amd64.tar.gz
```

3. 启动服务

```
cd prometheus-2.22.0.linux-amd64/
./prometheus --config.file=prometheus.yml
```

![image-20201104105749453](/Users/sunxiaojun/Library/Application Support/typora-user-images/image-20201104105749453.png)

4. 访问服务：http://127.0.0.1:9090

![image-20201104111554198](/Users/sunxiaojun/Library/Application Support/typora-user-images/image-20201104111554198.png)

### Prometheus Query Language

1. .* ：通配符，匹配0~n个任意字符
2. .   ：通配符，匹配任意一个字符
3. [30m:1m] ：返回过去30分钟的数据，以1分钟的粒度进行划分
4. =~ ：根据正则表达式匹配，过滤出匹配的数据
5. !~ ：根据正则表达式匹配过滤出不匹配的数据
6. offset 5m : 偏移量，统计5分钟之前的数据
7. expression language operateors：https://prometheus.io/docs/prometheus/latest/querying/operators/，包含**算术运算、比较运算、逻辑运算、向量匹配、聚合运算、优先级**
8. expression language functions：https://prometheus.io/docs/prometheus/latest/querying/functions/

### storage

1. 本地存储：本地存储只能当成单节点来使用，不支持高可用
2. 远程存储：通过HTTP存储到远程
   1. Influxdb：支持读/写操作（https://docs.influxdata.com/influxdb/v1.8/supported_protocols/prometheus/）
   2. Kafka：支持写数据到Kafka（https://github.com/Telefonica/prometheus-kafka-adapter）

### HTTP API

#### 返回体格式说明

在Prometheus服务器上的/ api / v1下可以访问当前稳定的HTTP API。API的响应格式为JSON，每个成功的API请求均返回2xx状态代码。

```
2xx : 查询成功，返回2xx
400 : 当参数丢失或不正确时,返回400错误
422 : 当遇到无法执行表达式时无法处理的实体时，返回422错误
503 : 服务在查询超时或中止时不可用,返回503错误
```

#### 表达式查询

**当遇到某些特殊字符不符合url规范的，在post方式时，可以通过设置*Content-Type: application/x-www-form-urlencoded*对URL进行编码。**

1. **及时查询接口（Instant queries）**

   接口地址：

   ```
   /api/v1/query （支持GET/POST方式）
   ```

   参数描述：

   ```
   query   :	String类型，Prometheus支持的表达式
   time	  : rfc3339格式/timestamp时间戳格式（可选参数）,精确到秒
   timeout : 超时时间（可选参数），缺省值为query.timeout，并以此为上限。
   ```

   示例：

   ```shell
   curl 'http://127.0.0.1:9090/api/v1/query?query=node_memory_Inactive_bytes'
   {
     "status": "success",
     "data": {
       "resultType": "vector",
       "result": [
         {
           "metric": {
             "__name__": "node_memory_Inactive_bytes",
             "group": "case",
             "instance": "localhost:8082",
             "job": "node"
           },
           "value": [
             1604901027.208,
             "892313600"
           ]
         },
         {
           "metric": {
             "__name__": "node_memory_Inactive_bytes",
             "group": "production",
             "instance": "localhost:8080",
             "job": "node"
           },
           "value": [
             1604901027.208,
             "893497344"
           ]
         },
         {
           "metric": {
             "__name__": "node_memory_Inactive_bytes",
             "group": "production",
             "instance": "localhost:8081",
             "job": "node"
           },
           "value": [
             1604901027.208,
             "893497344"
           ]
         }
       ]
     }
   }
   ```

   

2. **范围查询接口（Range queries）**

   接口地址：

   ```
   /api/v1/query_range	（支持GET/POST方式）
   ```

   参数描述:

   ```
   query   :	String类型，Prometheus支持的表达式
   start   : 起始时间（时间戳,精确到秒）
   end	    : 截止时间（时间戳,精确到秒）
   step    : 步长（持续时间格式或浮点秒数）
   timeout : 超时时间（可选参数），缺省值为query.timeout，并以此为上限。
   ```

   示例：

   ```shell
   curl 'http://127.0.0.1:9090/api/v1/query_range?query=node_memory_Inactive_bytes&start=1604902240&end=1604902480&step=15s'
   {
     "status": "success",
     "data": {
       "resultType": "matrix",
       "result": [
         {
           "metric": {
             "__name__": "node_memory_Inactive_bytes",
             "group": "case",
             "instance": "localhost:8082",
             "job": "node"
           },
           "values": [
             [
               1604902240,
               "893370368"
             ],
             [
               1604902255,
               "893468672"
             ],
             [
               1604902270,
               "893546496"
             ],
             [
               1604902285,
               "893644800"
             ],
             [
               1604902300,
               "893726720"
             ],
             [
               1604902315,
               "893825024"
             ],
             [
               1604902330,
               "893902848"
             ],
             [
               1604902345,
               "894001152"
             ],
             [
               1604902360,
               "894083072"
             ],
             [
               1604902375,
               "894177280"
             ],
             [
               1604902390,
               "894259200"
             ],
             [
               1604902405,
               "894353408"
             ],
             [
               1604902420,
               "894443520"
             ],
             [
               1604902435,
               "894533632"
             ],
             [
               1604902450,
               "894619648"
             ],
             [
               1604902465,
               "894709760"
             ],
             [
               1604902480,
               "894799872"
             ]
           ]
         },
         {
           "metric": {
             "__name__": "node_memory_Inactive_bytes",
             "group": "production",
             "instance": "localhost:8080",
             "job": "node"
           },
           "values": [
             [
               1604902240,
               "893390848"
             ],
             [
               1604902255,
               "893485056"
             ],
             [
               1604902270,
               "893575168"
             ],
             [
               1604902285,
               "893665280"
             ],
             [
               1604902300,
               "893755392"
             ],
             [
               1604902315,
               "893841408"
             ],
             [
               1604902330,
               "893931520"
             ],
             [
               1604902345,
               "894021632"
             ],
             [
               1604902360,
               "894111744"
             ],
             [
               1604902375,
               "894193664"
             ],
             [
               1604902390,
               "894291968"
             ],
             [
               1604902405,
               "894373888"
             ],
             [
               1604902420,
               "894468096"
             ],
             [
               1604902435,
               "894550016"
             ],
             [
               1604902450,
               "894644224"
             ],
             [
               1604902465,
               "894730240"
             ],
             [
               1604902480,
               "894824448"
             ]
           ]
         },
         {
           "metric": {
             "__name__": "node_memory_Inactive_bytes",
             "group": "production",
             "instance": "localhost:8081",
             "job": "node"
           },
           "values": [
             [
               1604902240,
               "893390848"
             ],
             [
               1604902255,
               "893485056"
             ],
             [
               1604902270,
               "893575168"
             ],
             [
               1604902285,
               "893661184"
             ],
             [
               1604902300,
               "893751296"
             ],
             [
               1604902315,
               "893841408"
             ],
             [
               1604902330,
               "893931520"
             ],
             [
               1604902345,
               "894017536"
             ],
             [
               1604902360,
               "894111744"
             ],
             [
               1604902375,
               "894193664"
             ],
             [
               1604902390,
               "894291968"
             ],
             [
               1604902405,
               "894369792"
             ],
             [
               1604902420,
               "894468096"
             ],
             [
               1604902435,
               "894550016"
             ],
             [
               1604902450,
               "894644224"
             ],
             [
               1604902465,
               "894726144"
             ],
             [
               1604902480,
               "894824448"
             ]
           ]
         }
       ]
     }
   }
   ```

3. **获取标签名称（Getting label names）**

   接口地址：

   ```
   /api/v1/labels	（支持GET/POST方式）
   ```

   参数描述：

   ```
   start   : 起始时间（时间戳,精确到秒，非必填）
   end	    : 截止时间（时间戳,精确到秒，非必填）
   ```

   示例：

   ```shell
   curl 'http://127.0.0.1:9090/api/v1/labels?start=1604902240&end=1604902480'
   {
     "status": "success",
     "data": [
       "__name__",
       "address",
       "branch",
       "broadcast",
       "call",
       "cause",
       "code",
       "collector",
       "config",
       "consumer",
       "cpu",
       "device",
       "dialer_name",
       "domainname",
       "duplex",
       "endpoint",
       "event",
       "fstype",
       "goversion",
       "group",
       "handler",
       "instance",
       "interval",
       "ip",
       "job",
       "le",
       "listener_name",
       "machine",
       "mode",
       "mountpoint",
       "name",
       "nodename",
       "operstate",
       "quantile",
       "queue",
       "reason",
       "release",
       "remote_name",
       "revision",
       "role",
       "rule_group",
       "scrape_job",
       "slice",
       "sysname",
       "type",
       "url",
       "version"
     ]
   }
   ```

4. **查询标签值（Querying label values）**

   接口地址：

   ```
   GET /api/v1/label/<label_name>/values
   ```

   参数描述：

   ```
   start   : 起始时间（时间戳,精确到秒，非必填）
   end	    : 截止时间（时间戳,精确到秒，非必填）
   ```

   示例：

   ```shell
   curl 'http://127.0.0.1:9090/api/v1/label/group/values'
   {
     "status": "success",
     "data": [
       "case",
       "production"
     ],
     "warnings": [
       "not implemented"
     ]
   }
   ```

#### 表达式查询结果格式（Expression query result formats）

**JSON格式数据**

#### 指标查询（Target）

接口地址：

```shell
GET /api/v1/targets  # 查询指标列表
```

参数说明：

```
state : 状态参数，枚举类型(active:激活, dropped:已删除, any:任何)
```

示例：

```shell
curl 'http://127.0.0.1:9090/api/v1/targets?state=active'
{
  "status": "success",
  "data": {
    "activeTargets": [
      {
        "discoveredLabels": {
          "__address__": "localhost:8082",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "group": "case",
          "job": "node"
        },
        "labels": {
          "group": "case",
          "instance": "localhost:8082",
          "job": "node"
        },
        "scrapePool": "node",
        "scrapeUrl": "http://localhost:8082/metrics",
        "globalUrl": "http://iZuf6278r1bks3obl3rr0hZ:8082/metrics",
        "lastError": "",
        "lastScrape": "2020-11-09T15:01:59.151159711+08:00",
        "lastScrapeDuration": 0.011007502,
        "health": "up"
      },
      {
        "discoveredLabels": {
          "__address__": "localhost:8080",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "group": "production",
          "job": "node"
        },
        "labels": {
          "group": "production",
          "instance": "localhost:8080",
          "job": "node"
        },
        "scrapePool": "node",
        "scrapeUrl": "http://localhost:8080/metrics",
        "globalUrl": "http://iZuf6278r1bks3obl3rr0hZ:8080/metrics",
        "lastError": "",
        "lastScrape": "2020-11-09T15:02:03.931152666+08:00",
        "lastScrapeDuration": 0.01526038,
        "health": "up"
      },
      {
        "discoveredLabels": {
          "__address__": "localhost:8081",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "group": "production",
          "job": "node"
        },
        "labels": {
          "group": "production",
          "instance": "localhost:8081",
          "job": "node"
        },
        "scrapePool": "node",
        "scrapeUrl": "http://localhost:8081/metrics",
        "globalUrl": "http://iZuf6278r1bks3obl3rr0hZ:8081/metrics",
        "lastError": "",
        "lastScrape": "2020-11-09T15:02:01.823718007+08:00",
        "lastScrapeDuration": 0.010659634,
        "health": "up"
      },
      {
        "discoveredLabels": {
          "__address__": "localhost:9090",
          "__metrics_path__": "/metrics",
          "__scheme__": "http",
          "job": "prometheus"
        },
        "labels": {
          "instance": "localhost:9090",
          "job": "prometheus"
        },
        "scrapePool": "prometheus",
        "scrapeUrl": "http://localhost:9090/metrics",
        "globalUrl": "http://iZuf6278r1bks3obl3rr0hZ:9090/metrics",
        "lastError": "",
        "lastScrape": "2020-11-09T15:02:04.253711483+08:00",
        "lastScrapeDuration": 0.007435956,
        "health": "up"
      }
    ],
    "droppedTargets": [
      
    ]
  }
}
```

#### 规则查询（Rules）

接口地址：

```shell
GET /api/v1/rules  # 查询规则列表
```

参数描述：

```shell
type : 规则类型，枚举值（alert:告警规则，record:记录规则） # 可选参数
```

示例：

```shell
curl 'http://127.0.0.1:9090/api/v1/rules'
{
  "status": "success",
  "data": {
    "groups": [
      {
        "name": "cpu-node",
        "file": "rules.yml",
        "rules": [
          {
            "name": "job_instance_mode:node_cpu_seconds:avg_rate5m",
            "query": "avg by(job, instance, mode) (rate(node_cpu_seconds_total[5m]))",
            "health": "ok",
            "evaluationTime": 0.000839683,
            "lastEvaluation": "2020-11-09T15:43:23.470159394+08:00",
            "type": "recording"
          }
        ],
        "interval": 15,
        "evaluationTime": 0.000850173,
        "lastEvaluation": "2020-11-09T15:43:23.470152599+08:00"
      }
    ]
  }
}
```

#### 告警查询（Alerts）

接口地址：

```shell
GET /api/v1/alerts  # 查询活动告警列表
```

示例：

```shell
curl 'http://127.0.0.1:9090/api/v1/alerts'
{
  "status": "success",
  "data": {
    "alerts": [
      
    ]
  }
}
```

#### 查询目标元数据（Querying target metadata）

接口地址：

```shell
GET /api/v1/targets/metadata  # 查询目标元数据，该接口后续可能会变化
```

参数描述：

```shell
match_target : 通过标签集匹配目标的标签选择器。如果保留为空，则选择所有目标。
metric : 要为其检索元数据的度量标准名称。如果保留为空，则将检索所有度量元数据。
limit : 匹配的最大目标数
```

#### 查询指标元数据（Querying metric metadata）

接口地址：

```shell
GET /api/v1/metadata # 查询指标元数据，该接口后续可能会变化		
```

参数描述：

```shell
metric : 要为其检索元数据的度量标准名称。如果保留为空，则将检索所有度量元数据。
limit : 匹配的最大目标数
```

#### 告警管理(Alertmanagers)

接口地址：

```shell
GET /api/v1/alertmanagers # 查询Prometheus可访问到的告警管理器	
```

#### 状态(status)

https://prometheus.io/docs/prometheus/latest/querying/api/#status

#### TSDB Admin APIs

https://prometheus.io/docs/prometheus/latest/querying/api/#tsdb-admin-apis

### 管理API

#### 健康检查

```shell
GET /-/healthy # 查询当前Prometheus的状态	

curl 'http://127.0.0.1:9090/-/healthy'

Prometheus is Healthy.
```

#### 准备检查

```shell
GET /-/ready  # 当Prometheus准备服务流量（即响应查询）时，此端点返回200。

curl 'http://127.0.0.1:9090/-/ready'

Prometheus is Ready.
```

#### 重载

默认情况下它是禁用的，可以通过**--web.enable-lifecycle**标志启用。另外，也可以通过向Prometheus进程发送SIGHUP来触发配置重载。

```shell
PUT/POST  /-/reload  #触发重载Prometheus配置和规则文件
```

#### 退出

默认情况下它是禁用的，可以通过**--web.enable-lifecycle**标志启用。另外，也可以通过将SIGTERM发送到Prometheus进程来触发正常关闭。

```shell
PUT/POST  /-/quit # 触发Prometheus的正常关闭
```

