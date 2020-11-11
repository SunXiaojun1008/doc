### 环境安装

#### 环境准备

**注：编译alertmanager时，需要使用go的环境变量，请先安装并配置好go的环境变量**

```shell
# 安装make命令及编译环境
yum -y install gcc automake autoconf libtool make

yum install gcc gcc-c++
```

#### 下载

```shell
mkdir -p $GOPATH/src/github.com/prometheus
cd $GOPATH/src/github.com/prometheus
wget https://codeload.github.com/prometheus/alertmanager/zip/v0.21.0
```

#### 编译安装

```shell
mv v0.21.0 v0.21.0.zip
unzip v0.21.0.zip
cd alertmanager-0.21.0
make build
# 编译成功后，会在当前目录下生成alertmanager可执行文件
```

### 服务启动

#### 配置文件编辑

对于Alertmanager不支持的通知机制，可使用webhook接收器进行集成（HTTP接口形式接收数据，POST请求方式）。此种方式适合于我们对告警数据进行二次处理后发送到监控平台。

```yml
global:
  resolve_timeout: 5m

route:
  group_by: ['node']
  group_wait: 30s
  group_interval: 30s
  repeat_interval: 30s
  receiver: webhook

receivers: 
- name: 'webhook'
  webhook_configs:
  - url: 'http://localhost:5000/receive'
```

自定义webhook接收器

```python
# -*- coding: UTF-8 -*-
__author__ = ''ahern
# 引入Flask框架
from flask import Flask,request
# 引入logging
import logging
# 引入json
import json
# 配置日志级别为DEBUG
logging.basicConfig(level=logging.DEBUG)

app = Flask(__name__)

# 定义数据接收接口
@app.route('/receive',methods = ['POST'])
def receive():
  try:
    data = json.loads(request.data)
    alerts =  data['alerts']
    # 后续处理（可自定义-> To Kafka/MQ...）
    for i in alerts:
      print('SEND SMS: ' + str(i))
  except Exception as e:
    print(e)
  return 'ok'
  #return 'My First Python Web Project! Hello 焕焕...'

if __name__ == "__main__":
    app.run()
```

```shell
SEND SMS: {'status': 'firing', 'labels': {'alertname': 'HighRequest', 'instance': 'localhost:8081', 'job': 'node', 'mode': 'idle'}, 'annotations': {}, 'startsAt': '2020-11-11T07:10:48.913980711Z', 'endsAt': '0001-01-01T00:00:00Z', 'generatorURL': 'http://iZuf6278r1bks3obl3rr0hZ:9090/graph?g0.expr=avg+by%28job%2C+instance%2C+mode%29+%28rate%28node_cpu_seconds_total%5B5m%5D%29%29+%3E+0.1&g0.tab=1', 'fingerprint': 'c36e458d2bd8b918'}
```



#### 服务启动

alertmanager启动时需要指定配置文件

```shell
./alertmanager --config.file=simple.yml #配置文件名称可自定义
```

