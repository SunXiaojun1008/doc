#### 查看类命令
##### 查看集群信息：kubectl cluster-info
```shell
[root@master ~]# kubectl cluster-info
Kubernetes master is running at https://192.168.56.107:6443
KubeDNS is running at https://192.168.56.107:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
##### 获取Pod相关信息：kubectl get pod
```shell
 # 查看默认NameSpace中的Pod
 kubectl get pods -n default

 # 查看指定NameSpace中的Pod
 kubectl get pod -n xxxxx（NameSpace）

 # 查看所有NameSpace的Pod
 kubectl get pod --all-namespaces

 # 查看Pod起在K8S集群的详细信息
 kubectl get pods -o wide

 # 查看Pod的日志
 kubectl logs -f xxxxxx(Pod名称)

 # Pod日志输出到文件
 kubectl logs xxxxxx(Pod名称) > ./log.log

 # Pod描述(查看Pod相关信息，Pod启动异常时，排错使用)
 kubectl describe pod xxxxxx(Pod名称)
 
 # 状态查询
 kubectl get po | grep opens
 
 # 按selector名来查找pod
 kubectl get pod --selector name=redis
```
##### 查看节点相关信息: kubectl get nodes
```shell
 #查看K8S集群节点信息信息
 kubectl get nodes -owide
 
 #查看某个节点详细信息
 kubectl describe node xxxxx
```

##### 查看命令空间相关信息: kubectl get namespace
```shell
 # 查看命名空间信息
 kubectl get namespace
 
 # 查看某个命名空间详细信息
 kubectl describe namespace xxxxxx
```

##### 查看服务相关信息：kubectl get service
```shell
 # 查看服务列表详细信息
 kubectl get service -o wide
 
 # 查看某个服务的明细信息（描述某个服务）
 kubectl describe service xxxxxx

```

##### configmap配置查询：kubectl get configmap
configmap：用于k8s集群中配置文件挂载，可以通过volumes挂载到容器内部
```shell
 # 查看configmap列表
 kubectl get configmap
 
 # 以yaml文件的格式查看configmap
 kubectl get configmap xxxxxx(configmap名称) -oyaml
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

#### 操作类命令

##### 创建资源
```shell
 # 创建资源
 kubectl create -f 文件名.yaml
 
 # 更新资源
 kubectl apply -f 文件名.yaml
 
 # 重建资源（--force ： 强制执行）
 kubectl replace -f 文件名  [--force]
```

##### 删除资源
```shell
 # 根据文件删除资源
 kubectl delete -f 文件名
 
 # 删除某个pod
 kubectl delete pod pod名
 
 # 删除某个rc
 kubectl delete rc rc名
 
 # 删除服务
 kubectl delete service service名
 
 # 删除全部pod
 kubectl delete pod --all
```

#### kubectl进阶命令操作
##### kubectl get：获取指定资源的基本信息
``` shell
 #查看所有service
 kubectl get services kubernetes-dashboard -n kube-system 
 
 #查看所有发布
 kubectl get deployment kubernetes-dashboard -n kube-system 
 
 #查看所有pod
 kubectl get pods --all-namespaces 
 
 #查看所有pod的IP及节点
 kubectl get pods -o wide --all-namespaces 
 
 # 带条件过滤的查询
 kubectl get pods -n kube-system | grep dashboard
```

##### kubectl scale：动态伸缩
```shell
 # 动态伸缩（rc、deployment）
 kubectl scale 资源类型 资源名称 --replicas=5 
 
 # 指定文件进行动态伸缩
 kubectl scale --replicas=2 -f 文件名.yaml #动态伸缩
```

##### kubectl exec：进入pod启动的容器
``` shell
 #进入容器 /bin/bash  || /bin/sh  || bash
 kubectl exec -it redis-master-1033017107-q47hh /bin/bash 
```

##### kubectl label：添加label值
可以通过增加label值的方式为节点增加污点，使启动的容器不落在当前节点上
```shell
 #增加节点lable值 spec.nodeSelector: zone: north #指定pod在哪个节点
 kubectl label nodes node1 zone=north
 
 #增加lable值 [key]=[value]
 kubectl label pod redis-master-1033017107-q47hh role=master 
 
 #删除lable值
 kubectl label pod redis-master-1033017107-q47hh role- 
 
 #修改lable值
 kubectl label pod redis-master-1033017107-q47hh role=backend --overwrite 

```

##### kubectl rolling-update：滚动升级
rolling-update:每启动一个新的容器就会删除一个旧的容器，使服务在不停服的情况下动态更新
``` shell
 #配置文件滚动升级
 kubectl rolling-update nginx -f 文件名.yaml 
 
 #命令升级（image镜像需要使用不同的版本）
 kubectl rolling-update nginx --image=image名称 

 #pod版本回滚
 kubectl rolling-update nginx --image=image名称 --rollback 
```


#### 帮助信息
1. kubectl -h 查看具体操作参数
```shell
[root@master ~]# kubectl -h
kubectl controls the Kubernetes cluster manager.

 Find more information at: https://kubernetes.io/docs/reference/kubectl/overview/

Basic Commands (Beginner):
  create         Create a resource from a file or from stdin.
  expose         使用 replication controller, service, deployment 或者 pod 并暴露它作为一个 新的
Kubernetes Service
  run            在集群中运行一个指定的镜像
  set            为 objects 设置一个指定的特征

Basic Commands (Intermediate):
  explain        查看资源的文档
  get            显示一个或更多 resources
  edit           在服务器上编辑一个资源
  delete         Delete resources by filenames, stdin, resources and names, or by resources and label selector

Deploy Commands:
  rollout        Manage the rollout of a resource
  scale          为 Deployment, ReplicaSet, Replication Controller 或者 Job 设置一个新的副本数量
  autoscale      自动调整一个 Deployment, ReplicaSet, 或者 ReplicationController 的副本数量

Cluster Management Commands:
  certificate    修改 certificate 资源.
  cluster-info   显示集群信息
  top            Display Resource (CPU/Memory/Storage) usage.
  cordon         标记 node 为 unschedulable
  uncordon       标记 node 为 schedulable
  drain          Drain node in preparation for maintenance
  taint          更新一个或者多个 node 上的 taints

Troubleshooting and Debugging Commands:
  describe       显示一个指定 resource 或者 group 的 resources 详情
  logs           输出容器在 pod 中的日志
  attach         Attach 到一个运行中的 container
  exec           在一个 container 中执行一个命令
  port-forward   Forward one or more local ports to a pod
  proxy          运行一个 proxy 到 Kubernetes API server
  cp             复制 files 和 directories 到 containers 和从容器中复制 files 和 directories.
  auth           Inspect authorization

Advanced Commands:
  diff           Diff live version against would-be applied version
  apply          通过文件名或标准输入流(stdin)对资源进行配置
  patch          使用 strategic merge patch 更新一个资源的 field(s)
  replace        通过 filename 或者 stdin替换一个资源
  wait           Experimental: Wait for a specific condition on one or many resources.
  convert        在不同的 API versions 转换配置文件
  kustomize      Build a kustomization target from a directory or a remote url.

Settings Commands:
  label          更新在这个资源上的 labels
  annotate       更新一个资源的注解
  completion     Output shell completion code for the specified shell (bash or zsh)

Other Commands:
  api-resources  Print the supported API resources on the server
  api-versions   Print the supported API versions on the server, in the form of "group/version"
  config         修改 kubeconfig 文件
  plugin         Provides utilities for interacting with plugins.
  version        输出 client 和 server 的版本信息

Usage:
  kubectl [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands).

```
2. 查看可操作资源
```shell

[root@master ~]# kubectl options
The following options can be passed to any command:

      --add-dir-header=false: If true, adds the file directory to the header
      --alsologtostderr=false: log to standard error as well as files
      --as='': Username to impersonate for the operation
      --as-group=[]: Group to impersonate for the operation, this flag can be repeated to specify multiple groups.
      --cache-dir='/root/.kube/http-cache': Default HTTP cache directory
      --certificate-authority='': Path to a cert file for the certificate authority
      --client-certificate='': Path to a client certificate file for TLS
      --client-key='': Path to a client key file for TLS
      --cluster='': The name of the kubeconfig cluster to use
      --context='': The name of the kubeconfig context to use
      --insecure-skip-tls-verify=false: If true, the server's certificate will not be checked for validity. This will
make your HTTPS connections insecure
      --kubeconfig='': Path to the kubeconfig file to use for CLI requests.
      --log-backtrace-at=:0: when logging hits line file:N, emit a stack trace
      --log-dir='': If non-empty, write log files in this directory
      --log-file='': If non-empty, use this log file
      --log-file-max-size=1800: Defines the maximum size a log file can grow to. Unit is megabytes. If the value is 0,
the maximum file size is unlimited.
      --log-flush-frequency=5s: Maximum number of seconds between log flushes
      --logtostderr=true: log to standard error instead of files
      --match-server-version=false: Require server version to match client version
  -n, --namespace='': If present, the namespace scope for this CLI request
      --password='': Password for basic authentication to the API server
      --profile='none': Name of profile to capture. One of (none|cpu|heap|goroutine|threadcreate|block|mutex)
      --profile-output='profile.pprof': Name of the file to write the profile to
      --request-timeout='0': The length of time to wait before giving up on a single server request. Non-zero values
should contain a corresponding time unit (e.g. 1s, 2m, 3h). A value of zero means don't timeout requests.
  -s, --server='': The address and port of the Kubernetes API server
      --skip-headers=false: If true, avoid header prefixes in the log messages
      --skip-log-headers=false: If true, avoid headers when opening log files
      --stderrthreshold=2: logs at or above this threshold go to stderr
      --token='': Bearer token for authentication to the API server
      --user='': The name of the kubeconfig user to use
      --username='': Username for basic authentication to the API server
  -v, --v=0: number for the log level verbosity
      --vmodule=: comma-separated list of pattern=N settings for file-filtered logging

```
3. 获取具体操作的帮助信息
```shell

[root@master ~]# kubectl get --help
Display one or many resources

 Prints a table of the most important information about the specified resources. You can filter the list using a label
selector and the --selector flag. If the desired resource type is namespaced you will only see results in your current
namespace unless you pass --all-namespaces.

 Uninitialized objects are not shown unless --include-uninitialized is passed.

 By specifying the output as 'template' and providing a Go template as the value of the --template flag, you can filter
the attributes of the fetched resources.

Use "kubectl api-resources" for a complete list of supported resources.

Examples:
  # List all pods in ps output format.
  kubectl get pods

  # List all pods in ps output format with more information (such as node name).
  kubectl get pods -o wide

  # List a single replication controller with specified NAME in ps output format.
  kubectl get replicationcontroller web

  # List deployments in JSON output format, in the "v1" version of the "apps" API group:
  kubectl get deployments.v1.apps -o json

  # List a single pod in JSON output format.
  kubectl get -o json pod web-pod-13je7

  # List a pod identified by type and name specified in "pod.yaml" in JSON output format.
  kubectl get -f pod.yaml -o json

  # List resources from a directory with kustomization.yaml - e.g. dir/kustomization.yaml.
  kubectl get -k dir/

  # Return only the phase value of the specified pod.
  kubectl get -o template pod/web-pod-13je7 --template={{.status.phase}}

  # List resource information in custom columns.
  kubectl get pod test-pod -o custom-columns=CONTAINER:.spec.containers[0].name,IMAGE:.spec.containers[0].image

  # List all replication controllers and services together in ps output format.
  kubectl get rc,services

  # List one or more resources by their type and names.
  kubectl get rc/web service/frontend pods/web-pod-13je7

Options:
  -A, --all-namespaces=false: If present, list the requested object(s) across all namespaces. Namespace in current
context is ignored even if specified with --namespace.
      --allow-missing-template-keys=true: If true, ignore any errors in templates when a field or map key is missing in
the template. Only applies to golang and jsonpath output formats.
      --chunk-size=500: Return large lists in chunks rather than all at once. Pass 0 to disable. This flag is beta and
may change in the future.
      --field-selector='': Selector (field query) to filter on, supports '=', '==', and '!='.(e.g. --field-selector
key1=value1,key2=value2). The server only supports a limited number of field queries per type.
  -f, --filename=[]: Filename, directory, or URL to files identifying the resource to get from a server.
      --ignore-not-found=false: If the requested object does not exist the command will return exit code 0.
  -k, --kustomize='': Process the kustomization directory. This flag can't be used together with -f or -R.
  -L, --label-columns=[]: Accepts a comma separated list of labels that are going to be presented as columns. Names are
case-sensitive. You can also use multiple flag options like -L label1 -L label2...
      --no-headers=false: When using the default or custom-column output format, don't print headers (default print
headers).
  -o, --output='': Output format. One of:
json|yaml|wide|name|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...
See custom columns [http://kubernetes.io/docs/user-guide/kubectl-overview/#custom-columns], golang template
[http://golang.org/pkg/text/template/#pkg-overview] and jsonpath template
[http://kubernetes.io/docs/user-guide/jsonpath].
      --output-watch-events=false: Output watch event objects when --watch or --watch-only is used. Existing objects are
output as initial ADDED events.
      --raw='': Raw URI to request from the server.  Uses the transport specified by the kubeconfig file.
  -R, --recursive=false: Process the directory used in -f, --filename recursively. Useful when you want to manage
related manifests organized within the same directory.
  -l, --selector='': Selector (label query) to filter on, supports '=', '==', and '!='.(e.g. -l key1=value1,key2=value2)
      --server-print=true: If true, have the server return the appropriate table output. Supports extension APIs and
CRDs.
      --show-kind=false: If present, list the resource type for the requested object(s).
      --show-labels=false: When printing, show all labels as the last column (default hide labels column)
      --sort-by='': If non-empty, sort list types using this field specification.  The field specification is expressed
as a JSONPath expression (e.g. '{.metadata.name}'). The field in the API resource specified by this JSONPath expression
must be an integer or a string.
      --template='': Template string or path to template file to use when -o=go-template, -o=go-template-file. The
template format is golang templates [http://golang.org/pkg/text/template/#pkg-overview].
  -w, --watch=false: After listing/getting the requested object, watch for changes. Uninitialized objects are excluded
if no object name is provided.
      --watch-only=false: Watch for changes to the requested object(s), without listing/getting first.

Usage:
  kubectl get
[(-o|--output=)json|yaml|wide|custom-columns=...|custom-columns-file=...|go-template=...|go-template-file=...|jsonpath=...|jsonpath-file=...]
(TYPE[.VERSION][.GROUP] [NAME | -l label] | TYPE[.VERSION][.GROUP]/NAME ...) [flags] [options]

Use "kubectl options" for a list of global command-line options (applies to all commands).

```