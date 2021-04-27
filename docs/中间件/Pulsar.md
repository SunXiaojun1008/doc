## Pulsar安装

### 前置

#### JDK

jdk的安装可以根据自己的需求进行安装

```shell
# 使用yum命令查询jdk
$ yum search java | grep jdk

# 使用yum命令安装jdk
$ yum install -y java-1.8.0-openjdk

# 查看java版本
$ java -version 

```

### Standalone

#### 下载

从官网下载pulsar安装文件

```shell
$ wget https://archive.apache.org/dist/pulsar/pulsar-2.7.0/apache-pulsar-2.7.0-bin.tar.gz
```

解压gz文件

```sh
$ tar xvfz apache-pulsar-2.7.0-bin.tar.gz
$ cd apache-pulsar-2.7.0
```

包内容说明

| 目录     | 内容                                                         |
| -------- | ------------------------------------------------------------ |
| bin      | pulsar的命令行工具，例如[`pulsar`](http://pulsar.apache.org/docs/en/reference-cli-tools#pulsar)和[`pulsar-admin`](http://pulsar.apache.org/docs/en/pulsar-admin) |
| conf     | pulsar的配置文件，包含broker configuration](http://pulsar.apache.org/docs/en/reference-configuration#broker),[ZooKeeper configuration](http://pulsar.apache.org/docs/en/reference-configuration#zookeeper)以及其他配置 |
| examples | java案例                                                     |
| lib      | pulsar依赖的jar包                                            |
| licenses | licenses key文件                                             |

在使用pulsar时，你需要自己创建一些文件夹

| 目录      | 内容                                          |
| --------- | --------------------------------------------- |
| data      | 这个目录用于存储Zookeeper以及BookKeeper的数据 |
| instances |                                               |
| logs      | 日志存放目录                                  |

#### 启动

```shell
$ bin/pulsar standalone
```

## Pulsar使用

