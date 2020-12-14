#### yaml文件

```shell
# 启动工程
kubectl apply -f ops-system-auth.yaml  
kubectl create -f ops-system-auth.yaml

# 删除工程
kubectl delete -f ops-system-auth.yaml
```

#### Pod

```shell
 # 查看默认NameSpace中的Pod
 kubectl get pods
 
 # 查看指定NameSpace中的Pod
 kubectl get pod -n xxxxx（NameSpace）
 
 # 查看所有NameSpace的Pod
 kubectl get pod --all-namespaces
 
 # 查看Pod起在K8S集群的哪个节点
 kubectl get pods -o wide
 
 # 删除Pod
 kubectl delete pod xxxxx(Pod名称)
 
 # 进入Pod内部
 kubectl exec -it xxxxxx(Pod名称) -- /bin/sh(/bin/bash)
 
 # 复制文件到容器内部的/path路径下
 kubectl cp file xxxxxx(Pod名称):/path/
 
 # 查看Pod的日志
 kubectl logs -f xxxxxx(Pod名称)
 
 # Pod日志输出到文件
 kubectl logs xxxxxx(Pod名称) > ./log.log
 
 # Pod描述(查看Pod相关信息，Pod启动异常时，排错使用)
 kubectl describe pod xxxxxx(Pod名称)
 
 # 状态查询
 kubectl get po | grep opens
 
 # 强制删除某个Pod
 kubectl delete pod xxxxxx(Pod名称) --grace-period=0 --force
 
 # 删除状态为Evicted的pod
 kubectl get pods | grep Evicted | awk '{print $1}' | xargs kubectl delete pod

 
```

#### NameSpace

```shell
# 查看当前K8S集群中的所有namespace
kubectl get namespace
```

#### Service

```shell
# 查看Service信息
kubectl get service
```

#### Nodes

```shell
# 查看K8S节点相关信息
kubectl get nodes -owide
```

#### ConfigMap

```shell
# 查询configMap列表
kubectl get configmap

# 以yaml文件的格式查询具体的configmap内容
kubectl get configmap xxxxx(configmap名称) -oyaml
```

#### K8S集群(K8S中的组件都基本是通过yaml的形式加载进去的)

```shell
# 查看K8S日志
journalctl -u kubelet -f

#查看K8S集群情况
kubectl get pod -n kube-system

# 查看K8S组件状态
kubectl get cs
```

