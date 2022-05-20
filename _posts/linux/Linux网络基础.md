## 1 Linux网络虚拟化

### 1.1Network Namespace

#### 1.1.1 初识Namespace

> 在Linux系统中，Namespace隔离技术很在便存在于内核中，并且Namespace就是为Linux的容器技术而设计的但是一直鲜为人知，直到Docker引领的容器技术革命爆发才慢慢进入大众的视野。

创建一个名为`netns1`的network namespace

```shell
# ip netns add netns1
```

当ip命令创建一个Network Namespace时，系统会在`/var/run/netns`路径下生成一个挂载点。挂载点的作用一方面是方便对Namespace的管理，另一方面是使Namespace及时没有进程运行也能继续存在。

```shell
# ls /var/run/netns/
netns1
```

通过`ip netns exec `命令进入Namespace，做一些网络查询/配置的工作：

> 1. `ip netns exec netns1`在netns1这个ns下执行命令
>
> 2. `ip link list` 需要执行的命令
>
>    整条命令就是进入netns1这个network Namespace查询网卡信息。

```shell
# ip netns exec netns1 ip link list
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

查看系统中有哪些network namespace可以使用以下命令：

```shell
# ip netns list 
netns1
```

如果想删除某个弃用的network  namespace可以使用以下命令：

```shell
# ip netns delete netns1
```

<font color="red">注意：上述命令实际上没有删除netns1这个net ns，只是移除了这个net ns对应的挂载点，只要里面还有进程在运行，net ns便会一直存在</font>

#### 1.1.2 配置Namespace

```shell
#进入netns1 这个net ns，ping 127.0.01
# ip netns exec netns1 ping 127.0.0.1
connect: 网络不可达
```

根据上述命令可以发现，一个新创建的net ns不包含任何网络设备，同时，自动的lo设备状态还是DOWN的。因为当尝试访问本地回环地址时，网络也是不通的。

如果我们想访问本地回环地址，首先需要将net ns中的lo设备状态设置为UP：

```shell
# ip netns exec netns1 ip link set dev lo up
# ip netns exec netns1 ip link list
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

此时，lo设备的状态已经被设置为UP，我们在尝试ping 127.0.0.01，让我们看看结果

```shell
# ip netns exec netns1 ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.021 ms
64 bytes from 127.0.0.1: icmp_seq=2 ttl=64 time=0.025 ms
64 bytes from 127.0.0.1: icmp_seq=3 ttl=64 time=0.024 ms
```

但是，仅一个本地回环设备是无法与外界通信的，如果我们想与外界通信，就需要在net ns上再创建一个虚拟的以太网卡（veth pair）。veth pair总是双向出现的并且相互连接的，就想linux的双向管道（pipe）。

创建一对虚拟以太网卡，然后将veth pair的一端放到netns1 net ns中：

```shell
# 创建一对虚拟以太网卡
# ip link add veth0 type veth peer name veth1

# 查看创建好的网卡
# ip link list | grep veth
5: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000
6: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000

# 将veth1放到 netns1 中
# ip link set veth1 netns netns1

# 再次查看veth pair列表
#  ip link list | grep veth
6: veth0@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT qlen 1000

```

此时，两块网卡间还不能通信，眼尖的同学可能发现了，此时两块网卡的状态都是DOWN，需要手动把状态设置为UP。

```shell
# 激活veth1，同时为其绑定ip：100.1.1.1
# ip netns exec netns1 ifconfig veth1 100.1.1.1/24 up

# 激活veth，同时为其绑定ip：100.1.1.2
#ifconfig veth0 100.1.1.2/24 up
```

上述命令执行完成后，此时我们就可以ping通 veth pair的任意一端了，例如在主机上ping 100.1.1.1：

```shell
# ping 100.1.1.1
PING 100.1.1.1 (100.1.1.1) 56(84) bytes of data.
64 bytes from 100.1.1.1: icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from 100.1.1.1: icmp_seq=2 ttl=64 time=0.030 ms
64 bytes from 100.1.1.1: icmp_seq=3 ttl=64 time=0.029 ms
```

另外，不通的net ns之间的路由表和防火墙规则等也是隔离的，因此我们刚刚创建的net ns也无法和主机共享路由表和防火墙规则。

```shell
# ip netns exec netns1 route
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
100.1.1.0       0.0.0.0         255.255.255.0   U     0      0        0 veth1

# ip netns exec netns1 iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination  
```

```shell
# ip netns exec netns1 ip link set veth1 netns 1
```

上面这条命令如何分解呢？

1. ip netns exec netns1 进入netns1 net ns。
2. ip link set veth1  netns 1 把netns1 net ns下的veth1网卡挪到pid为1的进程（即init进程）所在的net ns。

#### 1.1.3 Network Namespace API的使用

net ns作为Linux六大namespace之一，其API涉及三个Linux系统调佣：clone、unshare和setns，以及一些系统/proc目录下的文件。

clone()、unshare()、setns()系统调用会使用CLONE_NEW*常量来区别要操作的namespace类型。

> CLONE_NEWIPC、CLONE_NEWNS、CLONE_NEWNET、CLONE_NEWPID、CLONE_NEW USER和CLONE_NEWUTS

##### 1. clone系统调用（创建Namespace）

##### 2. /proc/PID/ns（维持Namespace存在）

> 每个Linux进程都拥有一个属于自己的/proc/PID/ns，这个目录下的每个文件都代表一个类型的namespace

```shell
# ls -l /proc/$$/ns
总用量 0
lrwxrwxrwx 1 root root 0 10月 28 16:09 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 10月 28 16:09 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 10月 28 16:09 net -> net:[4026531956]
lrwxrwxrwx 1 root root 0 10月 28 16:09 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 10月 28 16:09 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 10月 28 16:09 uts -> uts:[4026531838]
```

##### 3. setns系统调用（往ns中添加进程）

##### 4. unshare系统调用(将进程移除ns)

### 1.2 veth pair

> veth pair的创建和验证在上面我们已经测试过了，这里就不做过多赘述。

### 1.3 Linux bridge(网桥)

#### 1.3.1 创建我们的bridge

创建一个bridge：

```shell
# ip link add name br0 type bridge

# ip link set br0 up
```

为了充分发挥Linux bridge的作用，我们特将它和前文介绍的veth pair配合起来使用。我们将创建一对veth设备，并配置IP地址

```shell
# ip link add veth0 type veth peer name veth1
# ip addr add 1.2.3.101/24 dev veth0
# ip addr add 1.2.3.102/24 dev veth1
# ip link set veth0 up
# ip link set veth1 up
```

> **问题**：此时通过veth0 ping veth1 网卡，发现网络不通（实验结果如下），可以看到，由于 veth0 和 veth1 处于同一个网段，且是第一次连接，所以会事先发 ARP 包，但 veth1 并没有响应 ARP 包。
>
> ```shell
> ping -c 1 -I veth0 1.2.3.102
> PING 1.2.3.102 (1.2.3.102) from 1.2.3.101 veth0: 56(84) bytes of data.
> From 1.2.3.101 icmp_seq=1 Destination Host Unreachable
> 
> --- 1.2.3.102 ping statistics ---
> 1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
> ```
>
> 解决方案:
>
> 经查阅，这是由于我使用的 CentOS系统内核中一些 ARP 相关的默认配置限制所导致的，需要修改一下配置项：
>
> ```shell
> echo 1 > /proc/sys/net/ipv4/conf/veth1/accept_local
> echo 1 > /proc/sys/net/ipv4/conf/veth0/accept_local
> echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
> echo 0 > /proc/sys/net/ipv4/conf/veth0/rp_filter
> echo 0 > /proc/sys/net/ipv4/conf/veth1/rp_filter
> ```
>
> 修改完配置后，再次验证：
>
> ```shell
> # ping -c 1 -I veth0 1.2.3.102
> PING 1.2.3.102 (1.2.3.102) from 1.2.3.101 veth0: 56(84) bytes of data.
> 64 bytes from 1.2.3.102: icmp_seq=1 ttl=64 time=0.028 ms
> 
> --- 1.2.3.102 ping statistics ---
> 1 packets transmitted, 1 received, 0% packet loss, time 0ms
> rtt min/avg/max/mdev = 0.028/0.028/0.028/0.000 ms
> ```

将veth0绑定到br0上:

```shell
# ip link set dev veth0 master br0
```

通过bridge link 查看当前网桥上绑定了哪些网络设备:

```shell
# bridge link
9: veth0 state UP @veth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 master br0 state forwarding priority 32 cost 2 
```

br0和veth0相连之后发生了如下变化：

- br0和veth0之间连接起来了，并且是双向的通道；

- 协议栈和veth0之间变成了单通道，协议栈能发数据给veth0，但veth0从外面收到的数据不会转发给协议栈；

- br0的MAC地址变成了veth0的MAC地址。

这就好比Linux bridge在veth0和协议栈之间做了一次拦截，在veth0上面做了点小动作，将veth0本来要转发给协议栈的数据拦截，全部转发给bridge。同时，bridge也可以向veth0发数据。

```shell
# 从veth0 ping veth1
# ping -c 1  -I veth0 1.2.3.102
PING 1.2.3.102 (1.2.3.102) from 1.2.3.101 veth0: 56(84) bytes of data.
From 1.2.3.101 icmp_seq=1 Destination Host Unreachable

--- 1.2.3.102 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms

```

抓veth1的网卡报文：

```shell
# tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
17:07:51.965211 ARP, Request who-has 1.2.3.102 tell 1.2.3.101, length 28
17:07:52.966913 ARP, Request who-has 1.2.3.102 tell 1.2.3.101, length 28
17:07:53.969180 ARP, Request who-has 1.2.3.102 tell 1.2.3.101, length 28
```

抓veth0的网卡报文:

```shell
# tcpdump -n -i veth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
17:08:54.197129 ARP, Request who-has 1.2.3.102 tell 1.2.3.101, length 28
17:08:55.198936 ARP, Request who-has 1.2.3.102 tell 1.2.3.101, length 28
17:08:56.200877 ARP, Request who-has 1.2.3.102 tell 1.2.3.101, length 28
```

通过分析以下报文可以看出，包的去和回的流程都没有问题，问题就出在veth0收到应答包后没有给协议栈，而是给了br0，于是协议栈得不到veth1的MAC地址，导致通信失败

#### 1.3.2 把veth0的ip设置到br0上

> 通过上面的分析可以看出，给veth0配置IP没有意义，因为就算协议栈传数据包给veth0，回程报文也回不来。这里我们就把veth0的IP地址“让给”Linux bridge

```shell
# ip addr del 1.2.3.101/24 dev veth0

# ip addr add 1.2.3.101/24 dev br0
```

现在我们再通过br0去ping veth1，结果成功收到了ICMP的回程报文：

```shell
# ping -c 1 -I br0 1.2.3.102
PING 1.2.3.102 (1.2.3.102) from 1.2.3.101 br0: 56(84) bytes of data.
64 bytes from 1.2.3.102: icmp_seq=1 ttl=64 time=0.032 ms

--- 1.2.3.102 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.032/0.032/0.032/0.000 ms

```

此时，我们来尝试ping一下网关：

```shell
# ping -c 1 -I br0 1.2.3.1
PING 1.2.3.1 (1.2.3.1) from 1.2.3.101 br0: 56(84) bytes of data.
From 1.2.3.101 icmp_seq=1 Destination Host Unreachable

--- 1.2.3.1 ping statistics ---
1 packets transmitted, 0 received, +1 errors, 100% packet loss, time 0ms
```

如上所示，此时网关还无法ping通，因为在br0上只有1.2.3.101和1.2.3.102这两个网络设备。

#### 1.3.3 将物理网卡添加到Linux Bridge

接下来我们将主机上的物理网卡enp0s3添加到Linux Bridge

```shell
# ip link set dev enp0s3 master br0
```

此时，通过br0 ping 网关0.0.0.0，可以看到成功了：

```shell
# ping -c 1 -I br0 0.0.0.0
PING 0.0.0.0 (10.0.2.16) from 10.0.2.16 br0: 56(84) bytes of data.
64 bytes from 10.0.2.16: icmp_seq=1 ttl=64 time=0.017 ms

--- 0.0.0.0 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.017/0.017/0.017/0.000 ms
```

再来尝试通过veth1 ping 网关0.0.0.0:

```shell
# ping -c 1 -I veth1 0.0.0.0
PING 0.0.0.0 (10.0.2.17) from 10.0.2.17 veth1: 56(84) bytes of data.
64 bytes from 10.0.2.17: icmp_seq=1 ttl=64 time=0.018 ms

--- 0.0.0.0 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.018/0.018/0.018/0.000 ms

```

通过enp0s3 ping网关0.0.0.0:

```shell
```

### 1.4 tun/tap设备（用户态）

> tun/tap设备是理解flannel的基础，而flannel是一个重要的Kubernetes网络插件

从Linux文件系统的角度看，tun/tap设备是用户可以用文件句柄操作的字符设备；从网络虚拟化角度看，它是虚拟网卡，一端连着网络协议栈，另一端连着用户态程序。

tun表示虚拟的是点对点设备，tap表示虚拟的是以太网设备，这两种设备针对网络包实施不同的封装。

tun/tap设备的作用：tun/tap设备可以将TCP/IP协议栈处理好的网络包发送给任何一个使用tun/tap驱动的进程，由进程重新处理后发到物理链路中。

tun/tap设备其实就是利用Linux的设备文件实现内核态和用户态的数据交互，而访问设备文件则会调用设备驱动相应的例程，要知道设备驱动也是内核态和用户态的一个接口。

tap设备与tun设备的工作原理完全相同，区别在于：

- tun设备的/dev/tunX文件收发的是IP包，因此只能工作在L3，无法与物理网卡做桥接，但可以通过三层交换（例如ip_forward）与物理网卡连通；
- tap设备的/dev/tapX文件收发的是链路层数据包，可以与物理网卡做桥接。

创建一个Tun：

```c
#include <net/if.h>
#include <sys/ioctl.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <string.h>
#include <sys/types.h>
#include <linux/if_tun.h>
#include <stdlib.h>
#include <stdio.h>

int tun_alloc(int flags)
{
	struct ifreq ifr;
	int fd ,err;
	char *clonedev ="/dev/net/tun";
	if((fd =open(clonedev,O_RDWR))<0){
		return fd;
	}
	
	memset(&ifr,0,sizeof(ifr));
	ifr.ifr_flags=flags;
	if(( err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0 ){
		close(fd);
		return err;
	}
	printf("Open tun/tap device : %s for reading ...\n",ifr.ifr_name);
	return fd;
}
int main(){
	int tun_fd,nread;
	char buffer[1500];
	tun_fd = tun_alloc(IFF_TUN | IFF_NO_PI);
	if(tun_fd <0){
		perror("Allocating interface");
		exit(1);
	}
	while(1){
		nread = read(tun_fd, buffer, sizeof(buffer));
		if(nread<0){
			perror("Reading from interface");
			close(tun_fd);
			exit(1);
		}
		printf("Read %d bytes from tun/tap device \n",nread);
	}
	return 0;
}
```

### 1.5 iptables

> iptables的底层实现是netfilter。IP层的5个钩子点的位置，对应iptables就是5条内置链，分别是PREROUTING、POSTROUTING、INPUT、OUTPUT和FORWARD。

#### 1.5.1 netfilter

![netfilter原理图](..\images\linux\netfilter原理图.png)

当网卡上收到一个包送达协议栈时，最先经过的netfilter钩子是PREROUTING，如果确实有用户埋了这个钩子函数，那么内核将在这里对数据包进行目的地址转换（DNAT）。不管在PREROUTING有没有做过DNAT，内核都会通过查本地路由表决定这个数据包是发送给本地进程还是发送给其他机器。如果是发送给其他机器（或其他network namespace），就相当于把本地当作路由器，就会经过netfilter的FORWARD钩子，用户可以在此处设置包过滤钩子函数，例如iptables的reject函数。所有马上要发到协议栈外的包都会经过POSTROUTING钩子，用户可以在这里埋下源地址转换（SNAT）或源地址伪装（Masquerade，简称Masq）的钩子函数。如果经过上面的路由决策，内核决定把包发给本地进程，就会经过INPUT钩子。本地进程收到数据包后，回程报文会先经过OUTPUT钩子，然后经过一次路由决策（例如，决定从机器的哪块网卡出去，下一跳地址是多少等），最后出协议栈的网络包同样会经过POSTROUTING钩子。

Kubernetes网络之间用到的工具就有ebtables、iptables/ip6tables和conntrack，其中iptables是核心。

#### 1.5.2 table、chain和rule(iptables三板斧)

> iptables是用户空间的一个程序，通过netlink和内核的netfilter框架打交道，负责往钩子上配置回调函数。
>
> 一般情况下，iptables用户构建linux的防火墙，特殊情况下也可以做服务负载均衡（Kubernetes的Service就是基于iptables实现的负载均衡）。

![iptables工作原理](..\images\linux\iptables工作原理.png)

我们常说的iptables 5X5，即5张表（table）和5条链（chain）。5条链即iptables的5条内置链，对应上文介绍的netfilter的5个钩子。这5条链分别是：

- INPUT链：一般用于处理输入本地进程的数据包；
- OUTPUT链：一般用于处理本地进程的输出数据包；·
- FORWARD链：一般用于处理转发到其他机器/network namespace的数据包；
- PREROUTING链：可以在此处进行DNAT；
- POSTROUTING链：可以在此处进行SNAT。

除了系统预定义的5条iptables链，用户还可以在表中定义自己的链，5张表如下所示。这5张表的优先级从高到低是：raw、mangle、nat、filter、security。需要注意的是，iptables不支持用户自定义表。iptables的表用来分类管理iptables规则，系统所有的iptables规则都被划分到不同的表集合中。

- filter表：用于控制到达某条链上的数据包是继续放行、直接丢弃（drop）或拒绝（reject）；
- nat表：用于修改数据包的源和目的地址；
- mangle表：用于修改数据包的IP头信息；
- raw表：iptables是有状态的，即iptables对数据包有连接追踪（connectiontracking）机制，而raw是用来去除这种追踪机制的；
- security表：最不常用的表（通常，我们说iptables只有4张表，security表是新加入的特性），用于在数据包上应用SELinux。

iptables规则常见动作：

- DROP：直接将数据包丢弃，不再进行后续的处理。应用场景是不让某个数据源意识到你的系统的存在，可以用来模拟宕机；
- REJECT：给客户端返回一个connection refused或destination unreachable报文。应用场景是不让某个数据源访问你的系统，善意地告诉他：我这里没有你要的服务内容；
- QUEUE：将数据包放入用户空间的队列，供用户空间的程序处理；
- RETURN：跳出当前链，该链里后续的规则不再执行；
- ACCEPT：同意数据包通过，继续执行后续的规则；
- JUMP：跳转到其他用户自定义的链继续执行。

#### 1.5.3 ipbatles命令

- 查看所有iptables规则

  > iptables默认是输出filter表中的所有规则，若想输出指定表的规则使用`iptables -t nat -L -n`

  ```shell
  # iptables -L -n
  Chain INPUT (policy ACCEPT)
  target     prot opt source               destination         
  KUBE-NODEPORTS  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes health check service ports */
  KUBE-EXTERNAL-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes externally-visible service portals */
  KUBE-FIREWALL  all  --  0.0.0.0/0            0.0.0.0/0           
  
  Chain FORWARD (policy DROP)
  target     prot opt source               destination         
  KUBE-FORWARD  all  --  0.0.0.0/0            0.0.0.0/0            /* kubernetes forwarding rules */
  KUBE-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes service portals */
  KUBE-EXTERNAL-SERVICES  all  --  0.0.0.0/0            0.0.0.0/0            ctstate NEW /* kubernetes externally-visible service portals */
  DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0           
  DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0           
  ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
  DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
  ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
  ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
  ACCEPT     all  --  10.244.0.0/16        0.0.0.0/0           
  ACCEPT     all  --  0.0.0.0/0            10.244.0.0/16       
  ...
  ```

- 配置内置链的默认策略

  ```shell
  ## 默认的不让进
  # iptables --policy INPUT DROP
  
  ## 默认的不允许转发
  # intables -- policy FORWARD DROP
  
  ## 默认的可以出去
  # iptables --policy OUTPUT ACCEPT
  ```

- 配置防火墙规则策略

  - 配置允许SSH连接

    ```shell
    # iptables -A INPUT -s 192.168.1.1/24 -p tcp --dport 22 -j ACCEPT
    ```

    -A: 以追加的方式增加这条规则；

    -A INPUT：这条规则挂在INPUT链上；

    -s 192.168.1.1/24：表示允许源（source）地址是192.168.1.1/24这个网段的连接；

    -p tcp：表示允许TCP（protocol）包通过；

    --dport 22：允许访问的目的端口（destination port）为22；

    -j ACCEPT：接受这样的连接。

  - 阻止来自某个ip/网段的所有连接

    > -j DROP与-j REJECT的区别: 前者不会返回任何信息，只能等访问端超时推出；后者会发送一个连接拒绝的回程报文

    ```shell
    # iptables -A INPUT -s 192.168.1.1/24 -j DROP
    
    # iptables -A INPUT -s 192.168.1.1/24 -j REJECT
    ```

  - 封锁端口

    阻止本地端口8080建立对外的链接

    ```shell
    # iptables -A OUTPUT -p tcp --dport 8080 -j DROP
    ```

    阻止外部进程与本地8080建立链接

    ```shell
    # iptable -A INPUT -p tcp --dport 8080 -j DROP
    ```

  - 端口转发

    将访问80端口的请求转发到8080端口

    ```shell
    # iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j REDIRECT --to-port 8080
    ```

    -p tcp--dport 80-j REDIRECT--to-port 8080： 将发到TCP 80端口的流量转发（REDIRECT）给8080端口；

    -i eth0：表示匹配eth0网卡上接收到的包。

  - 禁用ping

    ```shell
    # iptables -A INPUT -p icmp DROP
    ```

    > - -p icmp-j DROP：表示匹配ICMP报文，然后丢弃。

  - 删除规则

    暴力删除本地所有规则

    ```shell
    # iptables -F
    ```

    清空特定表的规则使用参数-t

    ```shell
    # iptables -t nat -F 
    ```

    删除规则的最直接的需求是解封某条防火墙策略。因此，建议使用iptables的-D参数,-D表示从链中删除一条或多条指定的规则，后面跟的就是要删除的规则。

    ```shell
    # iptables -D INPUT -s 192.168.1.1 -j DROP
    ```

    当某条链上的规则被全部清除变成空链后，可以使用-X参数删除,删除FOO这条用户自定义的空链，但需要注意的是系统内置链无法删除。

    ```shell
    # iptables -X FOO
    ```

  - 自定义链

    我们在filter表（因为未指定表，所以可以使用-t参数指定）创建一条用户自定义的链BAR。

    ```shell
    # iptables -N BAR
    ```

- DNAT

  DNAT根据指定条件修改数据包的目标IP地址和目标端口。DNAT只发生在nat表的PREROUTING链，这也是我们要指定收到包的网卡而不是发出包的网卡的原因。

  ```shell
  # iptables -t nat -A PREROUTING -d 1.2.3.4 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.1:8080
  ```

  -j DNAT：表示目的地址转换；

  -d 1.2.3.4-p tcp--dport 80：表示匹配的包条件是访问目的地址和端口为1.2.3.4:80的TCP包；

  --to-destination：表示将该包的目的地址和端口修改成192.168.1.1:8080。

  注意：当涉及转发的目的IP地址是外机时，需要确保启用ip forward功能，即把Linux当交换机用,`echo 1 > /proc/sys/net/ipv4/ip_forward`

- SNAT（网络地址欺骗）

  SNAT根据指定条件修改数据包的源IP地址，即DNAT的逆操作。与DNAT的限制类似，SNAT策略只能发生在nat表的POSTROUTING链，与DNAT类似，我们也可以匹配发包的网卡，例如-o eth0（o是output的缩写）。

  ```shell
  # iptables -t nat -A PREROUTING -s 192.168.1.1 -j SNAT --to-source 172.16.230.98
  ```

  -j SNAT：表示源地址转换

  -s 192.168.1.1：表示匹配的包源地址是192.168.1.1

  --to-source：表示将该包的源地址修改成172.16.230.98

  至于网络地址伪装，其实就是一种特殊的源地址转换，报文从哪个网卡出就用该网卡上的IP地址替换该报文的源地址，具体用哪个IP地址由内核决定。与SNAT类似，如果要控制被替换的源地址，则可以指定匹配从哪块网卡发出报文。

  ```shell
  # iptables -t nat -A PREROUTING -s 192.168.1.1/24 -j MASQUERADE 
  ```

- 保存与回复

  之前的所有针对iptables规则的修改都是临时的，重启机器后就会丢失

  ```shell
  ## 永久保存
  # iptables-save
  
  ## 备份规则
  # iptables-save > iptables.bak
  
  ## 导入规则
  # iptables-save < iptables.bak
  ```



### 1.6 ipip

tun设备也叫作点对点设备，之所以叫这个名字，是因为tun经常被用来做隧道通信（tunnel）。tun设备也叫作点对点设备，之所以叫这个名字，是因为tun经常被用来做隧道通信（tunnel）。

```shell
# ip tunnel help
Usage: ip tunnel { add | change | del | show | prl | 6rd } [ NAME ]
          [ mode { ipip | gre | sit | isatap | vti } ] [ remote ADDR ] [ local ADDR ]
          [ [i|o]seq ] [ [i|o]key KEY ] [ [i|o]csum ]
          [ prl-default ADDR ] [ prl-nodefault ADDR ] [ prl-delete ADDR ]
          [ 6rd-prefix ADDR ] [ 6rd-relay_prefix ADDR ] [ 6rd-reset ]
          [ ttl TTL ] [ tos TOS ] [ [no]pmtudisc ] [ dev PHYS_DEV ]

Where: NAME := STRING
       ADDR := { IP_ADDRESS | any }
       TOS  := { STRING | 00..ff | inherit | inherit/STRING | inherit/00..ff }
       TTL  := { 1..255 | inherit }
       KEY  := { DOTTED_QUAD | NUMBER }
```

Linux原生支持5中L3隧道(Linux L3隧道底层实现原理都基于tun设备，因此我们可以将本节看作tun设备的高级应用):

- ipip：即IPv4 in IPv4，在IPv4报文的基础上封装一个IPv4报文；
- GRE：即通用路由封装（Generic Routing Encapsulation），定义了在任意一种网络层协议上封装其他任意一种网络层协议的机制，适用于IPv4和IPv6；
- sit：和ipip类似，不同的是sit用IPv4报文封装IPv6报文，即IPv6 over IPv4；
- ISATAP：即站内自动隧道寻址协议（Intra-Site Automatic Tunnel AddressingProtocol），与sit类似，也用于IPv6的隧道封装；
- VTI：即虚拟隧道接口（Virtual Tunnel Interface），是思科提出的一种IPSec隧道技术。下面我们以ipip为例，介绍Linux隧道通信的基本原理。

#### 1.6.1 ipip隧道

在使用ipip隧道前，需要Linux内核模块ipip.ko的支持。

通过lsmod|grep ipip查看内核是否加载，若没有则用modprobe ipip加载，正常加载应该显示：

```shell
#  modprobe ipip

#  lsmod |grep ipip
ipip                   16384  0 
tunnel4                16384  1 ipip
ip_tunnel              24576  1 ipip

```

创建隧道:

- 创建两个net ns 

  ```shell
  # ip netns add ns1
  # ip netns add ns2
  ```

- 创建两队veth pair，令其一段挂在某个ns下：

  ```shell
  # ip link add v1 type veth peer name v1_p
  # ip link add v2 type veth peer name v2_p
  
  # ip link set v1 netns ns1
  # ip link set v2 netns ns2
  ```

- 分别给两对veth-pair端点配上IP并启用:

  ```shell
  # ip addr add 10.10.10.1/24 dev v1_p
  # ip addr add 10.10.20.1/24 dev v2_p
  # ip link set v1_p up
  # ip link set v2_p up
  
  # ip netns exec ns1 ip addr add 10.10.10.2/24 dev v1
  # ip netns exec ns1 ip link set v1 up
  
  # ip netns exec ns2 ip addr add 10.10.20.2/24 dev v2
  # ip netns exec ns2 ip link set v2 up
  ```

- 验证v1 ping v2:

  ```shell
  # ip netns exec ns1 ping -c 1 -I v1 10.10.20.2
  PING 10.10.20.2 (10.10.20.2) from 10.10.10.2 v1: 56(84) bytes of data.
  
  --- 10.10.20.2 ping statistics ---
  1 packets transmitted, 0 received, 100% packet loss, time 0ms
  ```

  > 查看ip_forward的值
  >
  > ```shell
  > # cat /proc/sys/net/ipv4/ip_forward
  > 1
  > ```
  >
  > ip_forward选项已经打开，但是v1仍然ping不通v2
  >
  > 查看ns1的路由表
  >
  > ```shell
  > # ip netns exec ns1 route -n
  > Kernel IP routing table
  > Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  > 10.10.10.0      0.0.0.0         255.255.255.0   U     0      0        0 v1
  > ```
  >
  > 只有一条直连路由，没有通往10.10.20.0/24网段的路由，因此手动配置一条路由：
  >
  > ```shell
  > # ip netns exec ns1 route add -net 10.10.20.0 netmask 255.255.255.0 gw 10.10.10.1
  > 
  > ## 再次查看路由
  > # ip netns exec ns1 route -n
  > Kernel IP routing table
  > Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  > 10.10.10.0      0.0.0.0         255.255.255.0   U     0      0        0 v1
  > 10.10.20.0      10.10.10.1      255.255.255.0   UG    0      0        0 v1
  > 
  > # 同理，也给ns2配上通往10.10.10.0/24网段的路由
  > # ip netns exec ns2 route add -net 10.10.10.0 netmask 255.255.255.0 gw 10.10.20.1
  > 
  > # ip netns exec ns2 route -n
  > Kernel IP routing table
  > Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
  > 10.10.10.0      10.10.20.1      255.255.255.0   UG    0      0        0 v2
  > 10.10.20.0      0.0.0.0         255.255.255.0   U     0      0        0 v2
  > ```
  >
  > 再次验证v1 ping v2
  >
  > <font color="red">此处存在疑问，增加双向路由后，验证还是未通过</font>
  >
  > ```shell
  > # ip netns exec ns1 ping 10.10.20.2
  > ```
  >
  
- 创建tun1、tun2和ipip tunnel

  在ns1上创建tun1和ipip tunnel

  ```shell
  # ip netns exec ns1 ip tunnel add tun1 mode ipip remote 10.10.20.2 local 10.10.10.2
  # ip netns exec ns1 ip link set tun1 up
  # ip netns exec ns1 ip addr 10.10.100.10 peer 10.10.200.10 dev tun1
  ```

  在ns2上创建tun2和ipip tunnel

  ```shell
  # ip netns exec ns2 ip tunnel add tun2 mode ipip remote 10.10.10.2 local 10.10.20.2
  # ip netns exec ns2 ip link set tun2 up
  # ip netns exec ns2 ip addr 10.10.200.10 peer 10.10.100.10 dev tun2
  ```

实验过程：

- ping命令构建一个ICMP请求，ICMP报文封装在IP报文中，源和目的IP地址分别是10.10.100.10和10.10.200.10。
- 由于tun1和tun2不在同一网段，所以要查看路由表。通过ip tunnel命令建立ipip隧道后，会自动生成一条路由。
- 由于配置了隧道端点，数据包出了tun1，直接到达v1。根据ipip隧道的配置，会封装上一层新的IP头，源和目的IP地址分别为10.10.10.2和10.10.20.2。
- 由于v1和v2同样不在一个网段，查看路由表，发现去往10.10.20.0网段的报文从v1网卡出，去往10.10.10.1网关，即veth pair在主机上的另一端v1_p。
- Linux打开了ip_forward，它相当于一台路由器，10.10.10.0和10.10.20.0是两条直连路由，所以直接查路由表转发，从这台主机转到另一台主机的v2_p上。
- 根据veth pair的设备特性，数据包到达另一台主机的v2_p上，会直接从ns2的v2出来。内核解封装数据包，发现内层IP报文的目的IP地址是10.10.200.10，这正是自己配置的ipip隧道的tun2地址，于是将报文交给tun2设备。至此，tun1的ping请求包成功到达tun2。
- 由于ICMP报文的传输特性，有去必有回，所以ns2上会构造ICMP响应报文，并根据以上相同步骤封装和解封装数据包，直至到达tun1，整个ping过程完成。

### 1.7 vxlan

VXLAN（Virtual eXtensible LAN，虚拟可扩展的局域网），是一种虚拟化隧道通信技术。它是一种overlay（覆盖网络）技术，通过三层的网络搭建虚拟的二层网络

VXLAN是在底层物理网络（underlay）之上使用隧道技术，依托UDP层构建的overlay的逻辑网络，使逻辑网络与物理网络解耦，实现灵活的组网需求。它不仅能适配虚拟机环境，还能用于容器环境。由此可见，VXLAN这类隧道网络的一个特点是对原有的网络架构影响小，不需要对原网络做任何改动，就可在原网络的基础上架设一层新的网络。

#### 1.7.1 VXLAN协议原理

VXLAN的几个重要概念:

- VTEP（VXLAN Tunnel Endpoints）：VXLAN网络的边缘设备，用来进行VXLAN报文的处理（封包和解包）。VTEP可以是网络设备（例如交换机），也可以是一台机器（例如虚拟化集群中的宿主机）；
- VNI（VXLAN Network Identifier）：VNI是每个VXLAN的标识，是个24位整数，因此最大值是2的24次方（16777216）。如果一个VNI对应一个租户，那么理论上VXLAN可以支撑千万级别的租户。
- tunnel：隧道是一个逻辑上的概念，在VXLAN模型中并没有具体的物理实体相对应。隧道可以看作一种虚拟通道，VXLAN通信双方（图中的虚拟机）都认为自己在直接通信，并不知道底层网络的存在。从整体看，每个VXLAN网络像是为通信的虚拟机搭建了一个单独的通信通道，也就是隧道。

VXLAN的配置管理使用iproute2包，这个工具是和VXLAN一起合入内核的，我们常用的ip命令就是iproute2的客户端工具。VXLAN要求Linux内核版本在<font color="red">3.7</font>以上，最好为<font color="red">3.9以上</font>，所以在一些旧版本的Linux上无法使用基于VXLAN的封包技术。

#### 1.7.2 VXLAN组网

VXLAN报文的转发过程就是：原始报文经过VTEP，被Linux内核添加上VXLAN包头及外层的UDP头部，再发送出去，对端VTEP接收到VXLAN报文后拆除外层UDP头部，并根据VXLAN头部的VNI把原始报文发送到目的服务器。

VXLAN通信过程看似不复杂，但是前提是通信双方已经知道所有通信信息。第一次通信之前有以下问题待解决：

- 哪些VTEP需要加到一个相同的VNI组？

  > VTEP通常由网络管理员进行配置，所以哪些VTEP需要加到一个相同的VNI组由网络管理员决定。

- 发送方如何知道对方的MAC地址？

- 如何知道目的服务器在哪个节点上？

问题2、3可以归结为一个问题：VXLAN网络的通信双方如何感知彼此并选择正确的路径传输报文？要回答这两个问题，还得回到VXLAN协议报文上，看看一个完整的VXLAN报文需要哪些信息。

- 内层报文：通信双方的IP地址已经明确，需要VXLAN填充的是对方的MAC地址，VXLAN需要一个机制来实现ARP的功能；
- VXLAN头部：只需要知道VNI。它一般是直接配置在VTEP上的，即要么是提前规划的，要么是根据内部报文自动生成的；
- UDP头部：最重要的是源地址和目的地址的端口，源地址端口是由系统生成并管理的，目的端口一般固定为IANA分配的4789端口；
- IP头部：IP头部关心的是对端VTEP的IP地址，源地址可以用很简单的方式确定，目的地址是虚拟机所在地址宿主机VTEP的IP地址，需要由某种方式来确定；
- MAC头部：确定了VTEP的IP地址，MAC地址可以通过经典的ARP方式获取，毕竟VTEP在同一个三层网络内。

 总结一下，一个VXLAN报文需要确定两个地址信息：内层报文（对应目的虚拟机/容器）的MAC地址和外层报文（对应目的虚拟机/容器所在宿主机上的VTEP）IP地址。如果VNI也是动态感知的，那么VXLAN一共需要知道三个信息：内部MAC、VTEP IP和VNI。

一般有两种方式获得以上VXLAN网络的必要信息：多播和控制中心。多播的概念是同一个VXLAN网络的VTEP加入同一个多播网络。如果需要知道以上信息，就在组内发送多播来查询。控制中心的概念是在某个集中式的地方保存所有虚拟机的上述信息，自动告知VTEP它需要的信息即可。

参考资料：https://cizixs.com/2017/09/28/linux-vxlan/.

### 1.8 Macvlan

Macvlan的主要用途是网络虚拟化（包括容器和虚拟机）。另外，有一些比较特殊的场景，例如，keepalived使用虚拟MAC地址。需要注意的是，使用Macvlan的虚拟机或者容器网络与主机在同一个网段，即同一个广播域中。

Macvlan支持5种模式，分别是bridge、VEPA、Private、Passthru和Source模式。

Macvlan先天存在的不足:

- 无法支持大量的MAC地址；
- 无法工作在无线网络环境中。

### 1.9 IPvlan

与Macvlan类似，IPvlan也是从一个主机接口虚拟出多个虚拟网络接口。区别在于IPvlan所有的虚拟接口都有相同的MAC地址，而IP地址却各不相同。因为所有的IPvlan虚拟接口共享MAC地址，所以特别需要注意DHCP使用的场景。DHCP分配IP地址的时候一般会用MAC地址作为机器的标识。因此，在使用Macvlan的情况下，客户端动态获取IP的时候需要配置唯一的Client ID，并且DHCP服务器也要使用该字段作为机器标识，而不是使用MAC地址。

Linux内核<font color="red">3.19版本</font>才开始支持IPvlan，Docker从<font color="red">4.2版本</font>起能够稳定支持IPvlan。

IPvlan有两种不同的模式，分别是L2和L3。一个父接口只能选择其中一种模式，依附于它的所有子虚拟接口都运行在该模式下。

我们将IPvlan称为Macvlan的“救护员”是因为IPvlan除了能够完美解决以上问题，还允许用户基于IPvlan搭建比较复杂的网络拓扑，不再基于Macvlan的简单的二层网络，而是能够与BGP（Boader Gateway Protocol，边界网关协议）等协议扩展我们的网络边界。
