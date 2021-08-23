### 环境配置

#### Kerberos配置

Python进行Kerberos认证时，需要优先将认证的信息生成缓存文件:

```python
os.system('kinit -kt /etc/kafka.service.keytab kafka/ops0017@BCHKDC')
```

使用kinit命令则需要安装相关依赖包(Mac下已预装了Kerberos相关依赖包,可以忽略)

```shell
apt-get install krb5-kdc libkrb5-dev python3-six -y --fix-missing
```

安装完后，需要在/etc目录下创建一个krb5.conf,用来进行配置

```shell
[libdefaults]
  renew_lifetime = 7d
  forwardable = true
  default_realm = BCHKDC #此参数需要与Kerberos认证的描述文件相同：即上文的ops0017@BCHKDC的后缀
  ticket_lifetime = 24h
  dns_lookup_realm = false
  dns_lookup_kdc = false
  default_ccache_name = FILE:/tmp/krb5cc_%{uid}
  udp_preference_limit = 1
  default_tgs_enctypes = aes des3-cbc-sha1 rc4 des-cbc-md5
  #default_tkt_enctypes = aes des3-cbc-sha1 rc4 des-cbc-md5

[logging]
  default = FILE:/var/log/krb5kdc.log
  admin_server = FILE:/var/log/kadmind.log
  kdc = FILE:/var/log/krb5kdc.log

[realms]
  BCHKDC = {
    admin_server = ops0014 #此处填写Kerberos认证的地址域名（需要涉及host文件的配置） 
      kdc = ops0014

  }
```

#### Gssapi安装

使用pip命令安装gssapi

```shell
python -m pip install gssapi
```



### 配置文件

jaas.conf

```
Client {
   com.sun.security.auth.module.Krb5LoginModule required
   useKeyTab=true
   keyTab="/conf/kafka.service.keytab"
   storeKey=true
   useTicketCache=false
   serviceName="kafka"
   principal="kafka/ops0017@BCHKDC";
};
KafkaClient {
  com.sun.security.auth.module.Krb5LoginModule required
  useTicketCache=false
  useKeyTab=true
  principal="kafka/ops0017@BCHKDC"
  keyTab="/conf/kafka.service.keytab"
  storeKey=true
  serviceName="kafka";
};
```

### 代码编写

```python
# -*- coding:utf-8 -*-
__author__ = 'ahern

import os
import time
import json
from kafka import KafkaProducer
import logging
import gssapi
logging.basicConfig(level=logging.DEBUG)

def get_producer():

    # kafka鉴权文件软链接到/etc目录
    cur_dir = os.path.dirname(os.path.abspath(__file__))  # 自定义该目录
#    os.system('ln -fs %s/krb5.conf /etc/krb5.conf' % cur_dir)
#    os.system('ln -fs %s/kafka.service.keytab /etc/kafka.service.keytab' % cur_dir)
#    os.system('ln -fs %s/jaas.conf /etc/jaas.conf' % cur_dir)

    # 生成kafka认证密钥，配置系统环境变量
    os.system('kinit -kt /etc/kafka.service.keytab kafka/ops0017@BCHKDC')
    os.environ['KAFKA_OPTS'] = '-Djava.security.auth.login.config=/etc/jaas.conf -Djava.security.krb5.conf=/etc/krb5.conf'

    producer = KafkaProducer(**{
        'bootstrap_servers': ['ops0016:6667', 'ops0017:6667', 'ops0018:6667'],
        'security_protocol': 'SASL_PLAINTEXT',  # 安全协议
        'sasl_mechanism': 'GSSAPI',  # SASL机制
        'compression_type': 'gzip',  # 压缩方式，可选配置
        'api_version': (0, 10, 2),  # API版本
        'max_block_ms': 3000,  # 发送请求最大阻塞时间
        'value_serializer': lambda value: json.dumps(value).encode(),  # 数据序列化方法
        'sasl_kerberos_service_name': 'kafka'  # kerberos服务名称
    })

    # 等待0.5秒后检测是否连接成功了
    time.sleep(0.5)
    if not producer.bootstrap_connected():
        raise Exception('Connect kafka failed')
    return producer

# 因为底层socket的特性，多个进程或者同一进程下的多个线程无法共享一个producer，需要通过消息队列做生产者消费者模型
producer = get_producer()
print(producer.send('audit', {'test': 'test_data'}, partition=0).get(timeout=1))
```

