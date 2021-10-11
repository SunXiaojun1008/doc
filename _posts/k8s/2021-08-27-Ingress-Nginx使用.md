

### 版本选择

| Ingress-nginx version | k8s supported version  | Alpine Version | Nginx Version |
| --------------------- | ---------------------- | -------------- | ------------- |
| v1.0.0                | 1.22, 1.21, 1.20, 1.19 | 3.13.5         | 1.20.1        |
| v0.49.0               | 1.21, 1.20, 1.19       | 3.13.5         | 1.20.1        |
| v0.48.1               | 1.21, 1.20, 1.19       | 3.13.5         | 1.20.1        |
| v0.47.0               | 1.21, 1.20, 1.19       | 3.13.5         | 1.20.1        |
| v0.46.0               | 1.21, 1.20, 1.19       | 3.13.2         | 1.19.6        |

> 版本选择非常重要，目前我们K8s版本是1.18,顾选择v0.45.0以下版本

### ingress的部署

ingress的部署，需要考虑两个方面：

- ingress-controller是作为pod来运行的，以什么方式部署比较好？
- ingress解决了把如何请求路由到集群内部，那它自己怎么暴露给外部比较好？

#### Deployment+LoadBalancer模式的Service

> 如果要把ingress部署在公有云，那用这种方式比较合适。用Deployment部署ingress-controller，创建一个type为LoadBalancer的service关联这组pod。大部分公有云，都会为LoadBalancer的service自动创建一个负载均衡器，通常还绑定了公网地址。只要把域名解析指向该地址，就实现了集群服务的对外暴露。

#### Deployment+NodePort模式的Service

> 同样用deployment模式部署ingress-controller，并创建对应的服务，但是type为NodePort。这样，ingress就会暴露在集群节点ip的特定端口上。由于nodeport暴露的端口是随机端口，一般会在前面再搭建一套负载均衡器来转发请求。该方式一般用于宿主机是相对固定的环境ip地址不变的场景。
> <font color="red">注意：NodePort方式暴露ingress虽然简单方便，但是NodePort多了一层NAT，在请求量级很大时可能对性能会有一定影响。</font>

#### DaemonSet+HostNetwork+nodeSelector

> 用DaemonSet结合nodeselector来部署ingress-controller到特定的node上，然后使用HostNetwork直接把该pod与宿主机node的网络打通，直接使用宿主机的80/433端口就能访问服务。这时，ingress-controller所在的node机器就很类似传统架构的边缘节点，比如机房入口的nginx服务器。该方式整个请求链路最简单，性能相对NodePort模式更好。缺点是由于直接利用宿主机节点的网络和端口，一个node只能部署一个ingress-controller pod。比较适合大并发的生产环境使用。

### Ingress-nginx-controller安装

- 下载yaml文件

```shell
$ wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.44.0/deploy/static/provider/cloud/deploy.yaml
```

- 为节点增加标签

```shell
$ kubectl label node node01 isIngress="true"
```

- 修改`deploy.yaml`的配置

```yaml
# 修改api版本及kind
# apiVersion: apps/v1
# kind: Deployment
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
# 删除Replicas
# replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      # 选择对应标签的node
      nodeSelector:
        isIngress: "true"
      # 使用hostNetwork暴露服务
      hostNetwork: true
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
```

- 启动ingress-controller

> 若启动过成功出现`ImagePullError`的问题，可以先到阿里的镜像仓库中将镜像`k8s.gcr.io/ingress-nginx/controller:v0.44.0`下载到本地。

```shell
$ kubectl apply -f deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
daemonset.apps/ingress-nginx-controller created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created

# 检查部署情况
$ kubectl get daemonset -n ingress-nginx
NAME                       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                           AGE
ingress-nginx-controller   1         1         1       1            1           isIngress=true,kubernetes.io/os=linux   90s

$ kubectl get po -n ingress-nginx -o wide
NAME                                   READY   STATUS      RESTARTS   AGE    IP              NODE       NOMINATED NODE   READINESS GATES
ingress-nginx-admission-create-lkvpd   0/1     Completed   0          108s   10.222.51.206   node02   <none>           <none>
ingress-nginx-admission-patch-wk9c4    0/1     Completed   1          108s   10.222.51.234   node02   <none>           <none>
ingress-nginx-controller-hmmnk         1/1     Running     0          109s   10.154.5.55     node01   <none>           <none>

```

可以看到，Ingress-Nginx-Controller的服务已经已DaemonSet的资源启动在节点node01上。

```shell
$ netstat -lntup | grep nginx
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      429877/nginx: maste 
tcp        0      0 0.0.0.0:8181            0.0.0.0:*               LISTEN      429877/nginx: maste 
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      429877/nginx: maste 
tcp        0      0 127.0.0.1:10245         0.0.0.0:*               LISTEN      429616/nginx-ingres 
tcp        0      0 127.0.0.1:10246         0.0.0.0:*               LISTEN      429877/nginx: maste 
tcp        0      0 127.0.0.1:10247         0.0.0.0:*               LISTEN      429877/nginx: maste 
tcp6       0      0 :::10254                :::*                    LISTEN      429616/nginx-ingres 
tcp6       0      0 :::80                   :::*                    LISTEN      429877/nginx: maste 
tcp6       0      0 :::8181                 :::*                    LISTEN      429877/nginx: maste 
tcp6       0      0 :::443                  :::*                    LISTEN      429877/nginx: maste 

```

由于配置了hostnetwork，nginx已经在node主机本地监听80/443/8181端口。其中8181是nginx-controller默认配置的一个default backend。这样，只要访问node主机有公网IP，就可以直接映射域名来对外网暴露服务了。如果要nginx高可用的话，可以在多个node上部署，并在前面再搭建一套LVS+keepalive做负载均衡。用hostnetwork的另一个好处是，如果lvs用DR模式的话，是不支持端口映射的，这时候如果用nodeport，暴露非标准的端口，管理起来会很麻烦。

- 配置ingress资源

  > 部署完ingress-controller，接下来就按照测试的需求来创建ingress资源

  ```yaml
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: basic-ingress
    namespace: default
  spec:
    rules:
    - host: basic.ahern.com
      http:
        paths:
        - path: /
          backend:
            serviceName: ahern-system-basic
            servicePort: 8007
  ```

  ```shell
  $ kubectl apply -f basic-ingress.yaml
  ```

### 异常情况

- 创建ingress资源失败`ailed calling webhook "validate.nginx.ingress.kubernetes.io"`

  异常日志

```shell
[root@master ingress]# kubectl apply -f basic-ingress.yaml 

Error from server (InternalError): error when creating "basic-ingress.yaml": Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io": Post https://ingress-nginx-controller-admission.ingress-nginx.svc:443/networking/v1beta1/ingresses?timeout=10s: x509: certificate is not valid for any names, but wanted to match ingress-nginx-controller-admission.ingress-nginx.svc
```

​	根因分析：

> 从异常问题上看，是在创建Ingress资源时，验证未通过

​	解决方案

> 修改`ValidatingWebhookConfiguration`设置apiGroups范围

```yaml
# Source: ingress-nginx/templates/admission-webhooks/validating-webhook.yaml
# before changing this value, check the required kubernetes version
# https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#prerequisites
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  labels:
    helm.sh/chart: ingress-nginx-3.23.0
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 0.44.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: admission-webhook
  name: ingress-nginx-admission
webhooks:
  - name: validate.nginx.ingress.kubernetes.io
    matchPolicy: Equivalent
    rules:
      - apiGroups:
          - "" # 修改为""，全局可用
        apiVersions:
          - v1beta1
        operations:
          - CREATE
          - UPDATE
        resources:
          - ingresses
    failurePolicy: Fail
    sideEffects: None
    admissionReviewVersions:
      - v1
      - v1beta1
    clientConfig:
      service:
        namespace: ingress-nginx
        name: ingress-nginx-controller-admission
        path: /networking/v1beta1/ingresses
```

### 配置TCP/UDP

- 创建tcp-services.yaml文件，内容为下，以Redis为例

  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: tcp-services
    namespace: ingress-nginx
  data:
    6379: "default/redis-primary:6379"
  ```

  

- 创建configmap

  ```shell
  kubectl create -f tcp-services.yaml
  # 如果已经存在的话，直接编辑，在data节点下直接添加 port:namespace/service:port
  ```

- 修改`deploy.yaml`文件

  - 增加tcp-services-configmap配置（Deployment/DaemonSet）

  ```yaml
  - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services # 新增配置
  - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
  ```

  - 增加端口号

  ```yaml
    - name: redis-primary
      # nodePort: 36379  nodePort可以不用指定，系统自动生成
      port: 6379
      protocol: TCP
      targetPort: 2379
  ```

- reload ingress-nginx-controller

  ```shell
  kubectl apply -f deploy.yaml
  ```

  

