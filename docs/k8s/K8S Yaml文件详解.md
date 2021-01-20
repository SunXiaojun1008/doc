#### YAML基础

YAML是专门用来写配置文件的语言，非常简洁和强大，使用比json更方便。它实质上是一种通用的数据串行化格式。

YAML语法规则：

```
大小写敏感
使用缩进表示层级关系
缩进时不允许使用Tal键，只允许使用空格
缩进的空格数目不重要，只要相同层级的元素左侧对齐即可
”#” 表示注释，从这个字符一直到行尾，都会被解析器忽略　
“---”为可选的分隔符 ，当需要在一个文件中定义多个结构的时候需要使用
在Kubernetes中，只需要知道两种结构类型即可：
Lists
Maps
```

#### YMAL配置介绍

k8s的yaml文件包括四大组成部分，分别是：apiVersion、kind、metadata、spec

1. **apiVersion：** 这个版本号需要根据安装的K8S版本和资源类型进行变化，可以通过`kubecetl api-versions`进行查询。

   ```shell
   [root@ops0008 guard]# kubectl api-versions
   admissionregistration.k8s.io/v1
   admissionregistration.k8s.io/v1beta1
   apiextensions.k8s.io/v1
   apiextensions.k8s.io/v1beta1
   apiregistration.k8s.io/v1
   apiregistration.k8s.io/v1beta1
   apps/v1
   authentication.k8s.io/v1
   v1
   ...
   ```

2. **kind：**指定创建资源的角色/类型。常用类型有：**Namespace、Deployment、Service、Pod、Job、Ingress**等，根据不同的Kind，spec会有不有的配置

3. **metadata：**资源的元数据，包含Pod的一些meta信息，比如name、namespace、labels等信息

   - **name：** 资源的名字，在同一个namespace中必须唯一
   - **labels：**资源的标签

4. **spec：**包括一些container，storage，volume以及其他Kubernetes需要的参数，以及诸如是否在容器失败时重新启动容器的属性。可在特定Kubernetes API找到完整的Kubernetes Pod的属性。

#### Deployment常用配置

```yaml
apiVersion: apps/v1 #指定api版本
kind: Deployment #资源类型，
metadata: #资源的元数据/属性   
  name: ops-fms-rule # 资源的名字，在同一个namespace中必须唯一
  labels:  # 设定资源的标签
    app: fms-rule # 给ops-fms-rule资源增加一个app标签，value为fms-rule
spec: # 特定资源的详细内容
  replicas: 1 # 副本数
  strategy:  # 策略
    type: Recreate # 策略类型： Recreate 重新创建  RollingUpdate 滚动更新策略
  selector: # 选择器
    matchLabels:
      app: fms-rule
  template: # 模板
    metadata: # 资源的元数据/属性
      labels: # 设定资源的标签
        app: fms-rule
    spec: # 资源规范字段
      hostNetwork: true #使用主机网络
      hostAliases: # host 别名（域名配置）
      - ip: "10.154.5.210"
        hostnames:
        - "ops0014"
      containers: # 容器相关信息
      - name: ops-fms-rule # 容器的名字 
        image: registry.paas/ops/guard-fms-rule:2020-115 # 容器使用的镜像地址 
        imagePullPolicy: Always # 每次Pod启动拉取镜像策略，三个选择 Always、Never、IfNotPresent
                               # Always，每次都检查；Never，每次都不检查（不管本地是否有）；IfNotPresent，如果本地有就不检查，如果没有就拉取 
        resources: # 资源管理
          limits: # 最大使用
            cpu: 3000m # CPU，1核心 = 1000m
            memory: 2048Mi # 内存，1G = 1024Mi
          requests: # 容器运行时，最低资源需求，也就是说最少需要多少资源容器才能正常运行
            cpu: 3000m
            memory: 2048Mi
        volumeMounts: # 容器挂载
        - mountPath: /var/log/fms # 对应容器内部挂载的路径
          name: fms-rule-logs
        ports:
        - containerPort: 8002 # 容器开发对外的端口
          protocol: TCP # 协议
        env: # 环境变量
        - name: APOLLO_META
          value: http://10.154.8.25:18080
      restartPolicy: Always # 重启策略
      volumes: # 文件挂载
      - name: fms-rule-logs
        hostPath: # 主机挂载，
          path: /apps/logs/bc-ops/guard/fms-rule # 对应主机的path路径
      imagePullSecrets:  # 镜像仓库拉取密钥
      - name: harbor-secret
```

#### Service常用配置

在定义一个Service的时候，我们需要指定一个类型，如果不指定的话默认是ClusterIP类型。

可以使用的服务类型如下：

1. **NodePort**通过每个 Node 节点上的 IP 和静态端口（NodePort）暴露服务。NodePort 服务会路由到 ClusterIP 服务，这个 ClusterIP 服务会自动创建。通过请求 :，可以从集群的外部访问一个 NodePort 服务。（能够将内部服务暴露给外部的一种方式)。

2. **ClusterIP**通过集群的内部 IP 暴露服务(类似于一个VIP)，选择该值，服务只能够在集群内部可以访问，这也是默认的ServiceType。。

3. **LoadBalancer**功能与NodePort类似，区别在于 loadBalancer 比 nodePort 多了一步，使用云提供商的负载局衡器，可以向外部暴露服务。外部的负载均衡器可以路由到 NodePort 服务和 ClusterIP 服务，这个需要结合具体的云厂商进行操作。通俗说就是前面端口访问加了负载均衡
4. **ExternalName**通过返回 CNAME 和它的值，可以将服务映射到 externalName 字段的内容（例如， foo.bar.example.com）。没有任何类型代理被创建，这只有 Kubernetes 1.7 或更高版本的 kube-dns 才支持。

```yaml
apiVersion: v1 # 指定api版本，此值必须在kubectl api-versions中 
kind: Service # 指定创建资源的角色
metadata: # 资源的元数据/属性
  name: ops-fms-rule # 资源的名字，在同一个namespace中必须唯一
  labels: # 设定资源的标签
    app: fms-rule
spec: # 资源规范字段
  ports:
    - port: 8081 # service 端口
      targetPort: 8081 # 容器暴露的端口
      protocol: TCP # 协议
      nodePort: 30212 #Service对外暴露的端口
      name: http-port # 端口名称
  type: NodePort # Service类型 NodePort  ClusterIP
  selector: # 选择器 
    app: fms-rule # 表示此service需要关联包含标签app = fms-rule的pod上
    
```



