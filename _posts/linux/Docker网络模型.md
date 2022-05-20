## 2 Docker网络模型

### 2.1 Linux容器

容器不是模拟一个完整的操作系统，而是对进程进行隔离，隔离用到的技术就是Linux Namespace。对容器里的进程来说，它接触到的各种资源看似是独享的，但在底层其实是隔离的。容器是进程级别的隔离技术，因此相比虚拟机有启动快、占用资源少、体积小等优点。

### 2.2 Docker的四大网络模式

我们在使用docker run命令创建Docker容器时，可以使用--network选项指定容器的网络模式。Docker有以下4种网络模式：

- bridge模式，通过--network=bridge指定；
- host模式，通过--network=host指定；
- container模式，通过--network=container:NAME_or_ID指定，即joiner容器；
- none模式，通过--network=none指定。

### 2.3 常用的Docker网络技巧

#### 2.3.1 查看容器IP

```shell
# docker inspect mysql(<containerNameOrId>) | grep '"IPAddress"' | head -n 1

# docker inspect -f "{{ .NetworkSettings.IPAddress}}" mysql(<containerNameOrId>)
```

#### 2.3.2 端口映射

在使用docker run的时候可以使用-P或者-p命令进行容器和主机之间的端口映射。使用-P（大写）不需要指定任何映射关系，默认情况下，Docker会随机将一个4900049900的端口映射到内部容器开放的网络端口。使用-p（小写）则需要指定主机的端口应该映射到容器的哪个端口。

```shell
# docker run -p 1234:80 -d nginx
```

Docker容器端口映射原理都是在本地的iptable的nat表中添加相应的规则，将访问本机IP地址:hostport的网包进行一次DNAT，转换成容器IP:containerport。

```shell
# iptables -t nat -nL

# 查看端口映射情况 docker port container port
# docker port mysql 3306
```

#### 2.3.3 访问外网

一般情况下需要两个因素：ip_forward和SNAT/MASQUERADE。在默认情况下，容器可以访问外部网络的连接，因为容器的默认网络接口为docker0网桥上的接口，即主机上的本地接口。其原理是通过Linux系统的转发功能实现的（把主机当交换机）。如果发现容器内访问不了外网，则需要确认系统的ip_forward是否已打开。

或者检查docker daemon启动的时候--ip-forward参数是不是被设置成false了，如果是的话，则需要设置--ip-forward=true重新启动Docker，Docker会打开主机的ip forward。

至于SNAT/MASQUERADE，Docker会自动在iptables的POSTROUTING链上创建形如下面的规则:

```shell
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
KUBE-POSTROUTING  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes postrouting rules */
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0    
```

#### 2.3.4 DNS和主机名

容器中的DNS和主机名一般通过三个系统配置文件维护，分别是`/etc/resolv.conf`、`/etc/hosts`和`/etc/hostname`，其中:

- /etc/resolv/conf在创建容器的时候，默认与本地主机/etc/resolv.conf保持一致；
- /etc/hosts中则记载了容器自身的一些地址和名称；
- /etc/hostname中记录容器的主机名。

如果直接在容器内修改这三个文件会立即生效，但容器重启后修改又会失效。如果想统一、持久化配置所有容器的DNS，通过修改主机Docker Daemon的配置文件（一般是/etc/docker/daemon.json）的方式指定除主机/etc/resolv.conf的其他DNS信息：

```json
{
    "dns":{
        "114.114.114.114",
        "8.8.8.8"
    }
}
```

同时，也可以在docker run时使用--dns=address参数来指定。

至于配置Docker容器的主机名，则可以在运行docker run创建容器时使用参数-hhostname或者--hostname hostname配置。

#### 2.3.5 自定义网络

Docker默认会创建一个docker0的网桥，这个网桥和Docker容器使用的网段都是可以自定义的。

当启动Docker Daemon时，可以对docker0网桥进行配置

- --bip=CIDR,配置接到这个网桥上的Docker容器的IP地址网段，例如192.168.1.0/24；
- --mtu=BYTES,配置docker0网桥的MTU（最大传输单元）。

如果已经使用默认配置启动Docker Daemon，又希望有个自定义的容器网段却不想重启Docker Daemon，那么可以通过再创建一个自定义的网络达成目的。简而言之，一个主机上Docker支持多个网络。

Docker的命令行工具支持直接操作网络，就像操作容器一样。用户创建一个网络可以使用下面的命令：

```shell
#  docker network create -d bridge --subnet 172.25.0.0/16 mynet
9c680d88a082a89de9d0ea403a383c50e1f950f9ec0d0a1d5460d95b0e85d3fd
```

以上命令创建了一个名为mynet的网络，网络ID是9c680d88a082a...。该网络使用的是bridge模式，网段是172.25.0.0/16。我们可以通过 ip addr 查看一下这个网桥的名称：

```shell
# ip addr 
52: br-9c680d88a082: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:13:6b:c7:e5 brd ff:ff:ff:ff:ff:ff
    inet 172.25.0.1/16 brd 172.25.255.255 scope global br-9c680d88a082
       valid_lft forever preferred_lft forever
```

想查看某个网络的详细信息，可以使用inspect子命令，就像inspect容器一样。

```shell
#  docker inspect 9c680d88a082
[
    {
        "Name": "mynet",
        "Id": "9c680d88a082a89de9d0ea403a383c50e1f950f9ec0d0a1d5460d95b0e85d3fd",
        "Created": "2021-11-04T14:17:14.753386678+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.25.0.0/16"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```

查看主机上有多少个docker network，可以使用`docker network ls`命令:

```shell
#  docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
a8332d9d1da2        bridge              bridge              local
89e8d80d334f        host                host                local
9c680d88a082        mynet               bridge              local
966255e027a7        none                null                local
```

如果希望删除某个网络，则只需使用network rm子命令：

```shell
#  docker network rm 9c680d88a082
9c680d88a082
```

连接一个容器到网络中，用法如下:

```shell
#  docker network connect mynet <containerName>
```

将容器和网络断开:

```shell
# docker network disconnect
```

#### 2.3.6 发布服务

Docker的service子命令是对Docker容器的一层封装，有点类似于Kubernetes的Service概念。

```shell
# docker network create -d bridge foo 
# docker service create my-service.foo
#  docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                   PORTS
6hk9dgvp2klf        frosty_margulis     replicated          0/1                 my-service.foo:latest   

```



#### 2.3.7 Docker Link:两两互联（了解即可，不推荐使用）

docker link是一个遗留的特性，在新版本的Docker中，一般不推荐使用。

docker link就是把两个容器连起来，相互通信。

### 2.4 容器网络的第一个标准：CNM

围绕Docker生态，目前有两种主流的网络接口方案，即Docker主导的ContainerNetwork Model（CNM）和Kubernetes社区主推的Container NetworkInterface（CNI），其中CNM相对CNI较早提出。

CNM有3个主要概念：

- Network Sandbox：容器网络栈，包括网卡、路由表、DNS配置等。对应的技术实现有network namespace、FreeBSD Jail（类似于namespace的隔离技术）等。一个Sandbox可能包含来自多个网络（Network）的多个Endpoint；
- Endpoint：Endpoint作为Sandbox接入Network的介质，是Network Sandbox和Backend Network的中间桥梁。对应的技术实现有veth pair设备、tap/tun设备、OVS内部端口等；
- Backend Network：一组可以直接相互通信的Endpoint集合。对应的技术实现包括Linux bridge、VLAN等。如读者所见，一个Backend Network可以包含多个Endpoint。除此之外，CNM还需要依赖另外两个关键的对象来完成Docker的网络管理功能，它们分别是：
  - Network Controller：对外提供分配及管理网络的APIs，Docker Libnetwork支持多个网络驱动，Network Controller允许绑定特定的驱动到指定的网络；
  - Driver：网络驱动对用户而言是不直接交互的，它通过插件式的接入方式，提供最终网络功能的实现。Driver（包括IPAM）负责一个Network的管理，包括资源分配和回收。



### 2.5 容器组网挑战

一句话概括，容器网络的挑战：“以隔离的方式部署容器，在提供隔离自己容器内数据所需功能的同时，保持有效的连接性”，提炼两个关键词便是“隔离”和“连接”。虽然容器可以复用宿主机的基础网络“完美”解决“连接”问题，但网络隔离等问题决定了我们必须要为容器专门抽象出新的网络模型。



### 2.6 如何做好技术选型：容器组网方案沙场点兵

当前主流的容器网络虚拟化技术有Linux bridge、Macvlan、IPvlan、OpenvSwitch、flannel、Weave、Calico等。而容器网络最基础的一环是为容器分配IP地址，主流的方案有本地存储+固定网段的host-local，DHCP，分布式存储+IPAM的flannel和SDN的Weave等。

任何一个主流的容器组网方案无非就是网络虚拟机+IP地址分配，即Linuxbridge、Macvlan、IPvlan、Open vSwitch、flannel、Weave、Calico等虚拟化技术和host-local、DHCP、flannel、Weave任取两样的排列组合。

以Macvlan+DHCP为例，这种方案一般通过以下步骤初始化一个容器网络：

1. 创建Macvlan设备。
2. DHCP请求一个IP地址。
3. 为Macvlan设备配置IP地址。

而路由+IPAM初始化容器网络的步骤为：

1. 向IPAM请求一个IP地址。
2. 在主机和/或网络设备上创建veth设备和路由规则。
3. 为veth设备配置IP地址。

目前业界主流的容器解决方案大致可以分为“隧道方案”和“路由方案”，而这些方案其实都可以归结为容器网络的虚拟化方案。

#### 2.6.1 隧道方案

隧道网络也称为**overlay**网络，有时也被直译为覆盖网络。其实隧道方案不是容器领域新发明的概念，它在IaaS网络中应用得也比较多。overlay网络最大的优点是适用于几乎所有网络基础架构，它唯一的要求是主机之间的IP连接。但**overlay网络的问题是随着节点规模的增长，复杂度也会随之增加，而且用到了封包，因此出了网络问题定位起来比较麻烦**。

典型的基于overlay的网络插件有：Weave、Open vSwitch、flannel。

#### 2.6.2 路由方案

通过路由来实现，比较典型的网络插件有：Calico、Macvlan、Metaswitch

#### 2.6.3  容器网络标准接口

支持CNM标准的网络插件有： **Docker Swarm overlay**、**Macvlan&IP network drivers**、**Calico**、**Contiv**。

支持CNI标准的网络插件：**Weave**、**Macvlan**、**flannel**、**Calico**、**Contiv**、**Mesos CNI**。

CNI的优势是兼容其他容器技术（例如rkt）及上层编排系统（Kubernetes&Mesos），而且社区活跃势头迅猛，再加上Kubernetes主推，迅速成为容器网络的事实标准。
