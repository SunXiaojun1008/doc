## 3 Kubernetes网络原理与实践

### 3.1 Kubernetes网络

Kubernetes网络包括网络模型、CNI、Service、Ingress、DNS等。

在Kubernetes的网络模型中，每台服务器上的容器有自己独立的IP段，各个服务器之间的容器可以根据目标容器的IP地址进行访问。为了实现这一目的，重点解决IP地址分配和路由的问题。

#### 3.1.1 Kubernetes网络基础

**IP地址分配**

Kubernetes使用各种IP范围为节点、Pod和服务分配IP地址。

> - 系统会从集群的VPC网络为每个节点分配一个IP地址。该节点IP用于提供从控制组件（如Kube-proxy和Kubelet）到Kubernetes Master的连接；
> - 系统会为每个Pod分配一个地址块内的IP地址。用户可以选择在创建集群时通过--pod-cidr指定此范围；
> - 系统会从集群的VPC网络为每项服务分配一个IP地址（称为ClusterIP）。大部分情况下，该VPC与节点IP地址不在同一个网段，而且用户可以选择在创建集群时自定义VPC网络。

**Pod出站流量**

Kubernetes处理Pod的出站流量的方式主要分为以下三种:

- Pod to Pod

  在Kubernetes集群中，每个Pod都有自己的IP地址，运行在Pod内的应用都可以使用标准的端口号，不用重新映射到不同的随机端口号。所有的Pod之间都可以保持三层网络的连通性，比如可以相互ping对方，相互发送TCP/UDP/SCTP数据包。CNI就是用来实现这些网络功能的标准接口。

- Pod to Service

  Kubernetes通过Kube-proxy组件实现这些功能，每台计算节点上都运行一个Kubeproxy进程，通过复杂的iptables/IPVS规则在Pod和Service之间进行各种过滤和NAT。

- Pod to 集群外

  从Pod内部到集群外部的流量，Kubernetes会通过SNAT来处理。SNAT做的工作就是将数据包的源从Pod内部的IP:Port替换为宿主机的IP:Port。当数据包返回时，再将目的地址从宿主机的IP:Port替换为Pod内部的IP:Port，然后发送给Pod。当然，中间的整个过程对Pod来说是完全透明的，它们对地址转换不会有任何感知。

#### 3.2.2 Kubernetes主机内组网模型

Kubernetes经典的主机内组网模型是veth pair+bridge的方式。

当Kubernetes调度Pod在某个节点上运行时，它会在该节点的Linux内核中为Pod创建network namespace，供Pod内所有运行的容器使用。从容器的角度看，Pod是具有一个网络接口的物理机器，Pod中的所有容器都会看到此网络接口。因此，每个容器通过localhost就能访问同一个Pod内的其他容器。

Kubernetes使用veth pair将容器与主机的网络协议栈连接起来，从而使数据包可以进出Pod。容器放在主机根network namespace中veth pair的一端连接到Linux网桥，可让同一节点上的各Pod之间相互通信。

#### 3.2.3 Kubernetes跨节点组网模型

Kubernetes典型的跨机通信解决方案有bridge、overlay等。

- **bridge网络（Calico）**本身不解决容器的跨机通信问题，需要显式地书写主机路由表，映射目标容器网段和主机IP的关系，集群内如果有N个主机，需要N-1条路由表项。
- **overlay网络（flanneld）**，它是构建在物理网络之上的一个虚拟网络，其中VXLAN是主流的overlay标准。VXLAN就是用UDP包头封装二层帧，即所谓的MAC in UDP。

### 3.2 Pod的核心：pause容器

当检查Kubernetes集群的节点时，在节点上执行命令docker ps，用户可能会注意到一些被称为pause的容器。

Kubernetes中的pause容器可以说是网络模型的精髓，理解pause容器能够更好地理解Kubernetes Pod的设计初衷。

在Kubernetes中，pause容器被当作Pod中所有容器的“父容器”，并为每个业务容器提供以下功能：

- 在Pod中，它作为共享Linux namespace（Network、UTS等）的基础；
- 启用PID namespace共享，它为每个Pod提供1号进程，并收集Pod内的僵尸进程。

[pause源码](https://github.com/kubernetes/kubernetes/blob/master/build/pause/linux/pause.c):

```c
#include <signal.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

#define STRINGIFY(x) #x
#define VERSION_STRING(x) STRINGIFY(x)

#ifndef VERSION
#define VERSION HEAD
#endif

static void sigdown(int signo) {
  psignal(signo, "Shutting down, got signal");
  exit(0);
}

static void sigreap(int signo) {
  while (waitpid(-1, NULL, WNOHANG) > 0)
    ;
}

int main(int argc, char **argv) {
  int i;
  for (i = 1; i < argc; ++i) {
    if (!strcasecmp(argv[i], "-v")) {
      printf("pause.c %s\n", VERSION_STRING(VERSION));
      return 0;
    }
  }

  if (getpid() != 1)
    /* Not an error because pause sees use outside of infra containers. */
    fprintf(stderr, "Warning: pause should be the first process\n");

  if (sigaction(SIGINT, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 1;
  if (sigaction(SIGTERM, &(struct sigaction){.sa_handler = sigdown}, NULL) < 0)
    return 2;
  if (sigaction(SIGCHLD, &(struct sigaction){.sa_handler = sigreap,
                                             .sa_flags = SA_NOCLDSTOP},
                NULL) < 0)
    return 3;

  for (;;)
    pause();
  fprintf(stderr, "Error: infinite loop terminated\n");
  return 42;
}
```

pause容器运行一个非常简单的进程，**它不执行任何功能，一启动就永远把自己阻塞住了（见pause（）系统调用）**。正如你看到的，它当然不会只知道“睡觉”。它执行另一个重要的功能——即它**扮演PID 1的角色**，并在子进程成为“孤儿进程”的时候，**通过调用wait（）收割这些僵尸子进程**。这样就不用担心我们的Pod的PID namespace里会堆满僵尸进程了。

#### 3.2.1 从namespace 看pause

在Linux系统中运行新进程时，该进程从父进程继承了其namespace。在namespace中运行进程的方法是通过取消与父进程的共享namespace，从而创建一个新的namespace。

用户可以将其他进程添加到该进程的namespace中以形成一个Pod，Pod中的容器在其中共享namespace。

#### 3.2.2 从PID看pause容器

在容器中，必须要有一个进程充当每个PID namespace的init进程，使用Docker的话，ENTRYPOINT进程是init进程。如果多个容器之间共享PID namespace，那么拥有PID namespace的那个进程须承担init进程的角色，其他容器则作为init进程的子进程添加到PID namespace中。

#### 3.2.3 在Kubernetes中使用PID namespace共享/隔离

在Kubernetes 1.8版本之前，默认是启用PID namespace共享的，除非将Kubelet的标志--docker-disable-shared-pid设置成true，来禁用PID namespace共享。然而在Kubernetes 1.8版本以后，情况刚好相反，在默认情况下，Kubelet标志--docker-disable-sharedpid设置成true，如果要开启，还要设置成false。

我们知道Pod内容器共享PID namespace是很有意义的，那为什么还要开放这个禁止PID namesapce共享的开关呢？那是因为当应用程序不会产生其他进程，而且僵尸进程带来的问题可以忽略不计时，就用不到PIDnamespace的共享了。

在有些场景下，用户希望Pod内容器能够与其他容器隔离PID namespace，例如下面两个场景：

1. PID namespace共享时，由于pause容器成了PID=1，其他用户容器就没有PID 1了。但像systemd这类镜像要求获得PID 1，否则无法正常启动。有些容器通过kill-HUP 1命令重启进程，然而在由pause容器托管init进程的Pod里，上面这条命令只会给pause容器发信号。
2. PID namespace共享Pod内不同容器的进程对其他容器是可见的。这包括/proc中可见的所有信息，例如，作为参数或环境变量传递的密码，这将带来一定的安全风险。



