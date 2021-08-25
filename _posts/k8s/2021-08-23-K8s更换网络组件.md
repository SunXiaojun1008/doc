1. 停止现有组件

   ```shell
   kubectl delete -f calico.yaml
   ```

2. 删除网卡

   ```shell
   ifconfig cni0 down
   ip link delete cni0
   ifconfig flannel.1 down
   ip link delete flannel.1
   
   ifconfig tunl0 down
   ip link delete tunl0
   
   rm -rf /var/lib/cni/
   rm -rf /etc/cni/net.d/ 
   
   
   systemctl stop kubelet
   systemctl stop docker
   
   systemctl start docker
   systemctl start kubelet
   
   ```

4. 启动新的网络组件

   ```shell
   kubectl apply -f flannel.yaml
   ```




```
kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml

wget https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
vim calico.yaml
1）修改ipip模式关闭 和typha_service_name
- name: CALICO_IPV4POOL_IPIP
  value: "off"
typha_service_name: "calico-typha"
calico网络，默认是ipip模式（在每台node主机创建一个tunl0网口，这个隧道链接所有的node容器网络，官网推荐不同的ip网段适合，比如aws的不同区域主机），
修改成BGP模式，它会以daemonset方式安装在所有node主机，每台主机启动一个bird(BGP client)，它会将calico网络内的所有node分配的ip段告知集群内的主机，并通过本机的网卡eth0或者ens33转发数据；
2）修改replicas
replicas: 1
revisionHistoryLimit: 2
3）修改pod的网段CALICO_IPV4POOL_CIDR
- name: CALICO_IPV4POOL_CIDR
value: "192.168.0.0/16"
4）指定master网卡
在
- name: CLUSTER_TYPE
  value: "k8s,bgp"
下增加两行
- name: IP_AUTODETECTION_METHOD
  value: "interface=eth0"

kubectl apply -f calico.yaml
```

