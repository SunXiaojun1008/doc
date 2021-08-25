## 卸载K8s

### 1. 移除K8s节点

```shell
# 先执行以下命令将b-node2中的pod驱逐到别的可用节点上。
kubectl drain prometheus1 --delete-local-data --force --ignore-daemonsets
kubectl drain prometheus3 --delete-local-data --force --ignore-daemonsets

# 删除node
kubectl delete node prometheus1
kubectl delete node prometheus3

# kubeadm 重置节点
kubeadm reset -f

# 查找已安装的 kubeadm、kubectl、kubelet包
yum  list installed  | grep kubelet
yum  list installed  | grep kubeadm
yum  list installed  | grep kubectl
# 删除已安装的包
yum remove -y kubelet kubeadm kubectl
# 再次确认
```

### 2.移除Docker

```shell
# 停止Docker服务
systemctl stop docker 

# 查询docker安装过的包
yum  list installed | grep docker

# 删除安装包
yum remove -y docker-ce containerd.io docker-ce-cli

# 删除镜像/容器等
rm -rf /var/lib/docker
```

