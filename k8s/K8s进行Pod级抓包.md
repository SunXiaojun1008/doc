1. 列出待抓包的Pod及分布在哪些节点上

   > - -n dpm: 执行命名空间
   > - -o wide: 显示详细信息
   > - grep prometheus-: 筛选出包含`prometheus-`的记录

   ```shell
   [root@prometheus2 ~]#  kubectl get pod -n dpm -owide | grep prometheus-
   prometheus-0                                 2/2     Running   1          7m27s   192.168.117.76   prometheus3   <none>           <none>
   prometheus-node-0                            1/1     Running   0          8m11s   192.168.15.85    prometheus1   <none>           <none>
   prometheus-pulsar-adaptor-6d7d99c666-r2kzw   1/1     Running   0          13h     192.168.117.73   prometheus3   <none>           <none>
   ```

2. 找到所要抓包的Pod对应的containerID

   > `-o yaml`: 指定Pod的信息已yaml文件的格式输出，此处也可以使用`-o json`（指定以json文件的格式输出）
   >
   > 记录下我们找到的containerID: 149ceba29a1dde3ba46c2f7ea4c352e5033da54dd3b925fd155f8874376a16be

   ```shell
   [root@prometheus2 ~]#  kubectl get pod -n dpm prometheus-node-0 -o yaml
   apiVersion: v1
   kind: Pod
   metadata:
     ...
     containerStatuses:
     - containerID: docker://149ceba29a1dde3ba46c2f7ea4c352e5033da54dd3b925fd155f8874376a16be
       image: quay.io/prometheus/prometheus:v2.15.2
       imageID: docker-pullable://quay.io/prometheus/prometheus@sha256:914525123cf76a15a6aaeac069fcb445ce8fb125113d1bc5b15854bc1e8b6353
       lastState: {}
       name: prometheus-node
       ready: true
       restartCount: 0
       started: true
       state:
         running:
           startedAt: "2021-04-20T01:30:50Z"
     hostIP: 10.154.8.175
     phase: Running
     podIP: 192.168.15.85
     podIPs:
     - ip: 192.168.15.85
     qosClass: BestEffort
     startTime: "2021-04-20T01:30:48Z"
   ```

3. 到对应的K8s Node节点上执行下面命令

   > `/bin/sh`这个参数需要根据镜像的情况进行调整，有`/bin/sh`,`/bin/bash`,`/sh`,`/bash`等

   ```shell
   [root@prometheus1 ~]#  docker exec 149ceba29a1dde3ba46c2f7ea4c352e5033da54dd3b925fd155f8874376a16be /bin/sh -c 'cat /sys/class/net/eth0/iflink'
   174
   ```

4. 找对应的虚拟网卡地址

   > `/sys/class/net/cali*/ifindex`需要根据实际情况进行调整，可以先查看一下虚拟网卡的列表信息

   ```shell
   # 查看虚拟网卡列表
   [root@prometheus1 ~]#  ls /sys/class/net/
   cali02e26653b7a  cali549753fdbf7  cali7fcc33642a0  cali9bb6fba4d0d  docker0  lo
   cali4c026b1e165  cali78656a93d84  cali9a2b5f3d7f4  calie0fc1d4f147  eth0     tunl0
   ```

   ```shell
   # 找出对应的虚拟网卡
   [root@prometheus1 ~]#  for i in /sys/class/net/cali*/ifindex; do grep -l 174 $i; done
   /sys/class/net/calie0fc1d4f147/ifindex
   ```

5. 在宿主机上抓包

   ```shell
   [root@prometheus1 ~]#  tcpdump -i calie0fc1d4f147 -c 1000 -w /tmp/tcpdump.cap
   ```

6. 使用Wireshark分析

