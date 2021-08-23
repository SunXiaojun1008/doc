## Helm安装

1. 安装包下载

   从github中下载helm安装包[Releases Page](https://github.com/helm/helm/releases)

   > helm3 删除了tiiler

2. 解压安装包

   ```shell
   [root@root ~/helm]# tar -zxvf helm-v3.6.0-linux-amd64.tar.gz
   ```

3. 将解压后的文件复制到`/usr/local/bin/helm`目录

   ```shell
   [root@root ~/helm]# mv linux-adm64/helm /usr/local/bin/
   ```

4. 测试

   ```shell
   [root@root ~/helm]# helm help
   The Kubernetes package manager
   
   Common actions for Helm:
   
   - helm search:    search for charts
   - helm pull:      download a chart to your local directory to view
   - helm install:   upload the chart to Kubernetes
   - helm list:      list releases of charts
   
   Environment variables:
   ...
   
   ```

## 配置仓库

helm repo add apphub https://apphub.aliyuncs.com   ## apphub #仓库名 **https://apphub.aliyuncs.com** # 仓库地址

helm repo list # 查看仓库

helm repo update # 更新仓库 

helm repo remove baidu # 删除仓库

## HELM 部署应用

- 搜索应用

  Helm 自带一个强大的搜索命令，可以用来从两种来源中进行搜索

  - helm search hub nginx： 从Artifact Hub 中查找并列出helm charts；

  - helm search repo nginx：从你添加（使用 `helm repo add`）到本地 helm 客户端中的仓库中进行查找。该命令基于本地数据进行搜索，无需连接互联网。

  > Helm Search 使用模糊字符串匹配算法，所以如果你只记得部分名称，也可以进行搜索。

- 创建应用

  - 使用`helm create` 创建 charts模板，生成如下结构的文件夹

    ```shell
    [root@root ~/helm]# helm create nginx
        nginx
        |__ charts
        |__ Chart.yaml
        |__ templates
        	|__ deployment.yaml
        	|__ _helpers.tpl
        	|__ hpa.yaml
        	|__ ingress.yaml
        	|__ NOTES.txt
        	|__ serviceaccount.yaml
        	|__ service.yaml
        |__	values.yaml
    ```

  - 修改 values.yaml文件

    ```yaml
    # Default values for nginx.
    # This is a YAML-formatted file.
    # Declare variables to be passed into your templates.
    
    replicaCount: 1
    
    image:
      repository: nginx
      pullPolicy: IfNotPresent
      # Overrides the image tag whose default is the chart appVersion.
      tag: ""
    
    imagePullSecrets: []
    nameOverride: ""
    fullnameOverride: ""
    
    serviceAccount:
      # Specifies whether a service account should be created
      create: true
      # Annotations to add to the service account
      annotations: {}
      # The name of the service account to use.
      # If not set and create is true, a name is generated using the fullname template
      name: ""
    
    podAnnotations: {}
    
    podSecurityContext: {}
      # fsGroup: 2000
    
    securityContext: {}
      # capabilities:
      #   drop:
      #   - ALL
      # readOnlyRootFilesystem: true
      # runAsNonRoot: true
      # runAsUser: 1000
    
    service:
      type: ClusterIP
      port: 80
    
    ingress:
      enabled: false
      className: ""
      annotations: {}
        # kubernetes.io/ingress.class: nginx
        # kubernetes.io/tls-acme: "true"
      hosts:
        - host: chart-example.local
          paths:
            - path: /
              pathType: ImplementationSpecific
      tls: []
      #  - secretName: chart-example-tls
      #    hosts:
      #      - chart-example.local
    
    resources:
      # We usually recommend not to specify default resources and to leave this as a conscious
      # choice for the user. This also increases chances charts run on environments with little
      # resources, such as Minikube. If you do want to specify resources, uncomment the following
      # lines, adjust them as necessary, and remove the curly braces after 'resources:'.
      limits:
        cpu: 100m
        memory: 128Mi
      requests:
        cpu: 100m
        memory: 128Mi
    
    autoscaling:
      enabled: false
      minReplicas: 1
      maxReplicas: 100
      targetCPUUtilizationPercentage: 80
      # targetMemoryUtilizationPercentage: 80
    
    nodeSelector: {}
    
    tolerations: []
    
    affinity: {}
    
    ```

  - 验证文件是否符合规范

    ```shell
    [root@root ~/helm]# helm lint
    ```

  - 安装依赖

    使用`helm dependency build`构建所需依赖的charts,执行完成后会在当前目录下生成一个`charts`文件夹，包含所依赖的tgz后缀的文件包。

    ```shell
    [root@root ~/helm/nginx]#  helm dependency build
    Saving 1 charts
    Deleting outdated charts
    ```

- 安装应用

  - 使用 `helm install` 命令来安装一个新的 helm 包。最简单的使用方法只需要传入两个参数：你命名的release名字和你想安装的chart的名称

  ```shell
  [root@dpm01 ~/helm]#  helm install nginx --debug ./nginx/ --set service.type=NodePort
  install.go:173: [debug] Original chart version: ""
  install.go:190: [debug] CHART PATH: /root/helm/nginx
  
  client.go:122: [debug] creating 6 resource(s)
  NAME: nginx
  LAST DEPLOYED: Wed Aug 11 10:23:14 2021
  NAMESPACE: default
  STATUS: deployed
  REVISION: 1
  USER-SUPPLIED VALUES:
  service:
    type: NodePort
  ...
  NOTES:
  1. Get the application URL by running these commands:
    export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services nginx)
    export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT
  
  ```

  -  在安装过程中，`helm` 客户端会打印一些有用的信息，其中包括：哪些资源已经被创建，release当前的状态，以及你是否还需要执行额外的配置步骤。

  Helm按照以下顺序安装资源，若某类资源不存在，则会跳过继续安装下一个:

  ```
  - Namespace
  - NetworkPolicy
  - ResourceQuota
  - LimitRange
  - PodSecurityPolicy
  - PodDisruptionBudget
  - ServiceAccount
  - Secret
  - SecretList
  - ConfigMap
  - StorageClass
  - PersistentVolume
  - PersistentVolumeClaim
  - CustomResourceDefinition
  - ClusterRole
  - ClusterRoleList
  - ClusterRoleBinding
  - ClusterRoleBindingList
  - Role
  - RoleList
  - RoleBinding
  - RoleBindingList
  - Service
  - DaemonSet
  - Pod
  - ReplicationController
  - ReplicaSet
  - Deployment
  - HorizontalPodAutoscaler
  - StatefulSet
  - Job
  - CronJob
  - Ingress
  - APIService
  ```

  - 安装执行完成之后，可以通过`helm status`查看release分状态

  ```shell
  [root@root ~]#  helm status nginx
  NAME: nginx
  LAST DEPLOYED: Wed Aug 11 10:23:14 2021
  NAMESPACE: default
  STATUS: deployed
  REVISION: 1
  NOTES:
  1. Get the application URL by running these commands:
    export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services nginx)
    export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
    echo http://$NODE_IP:$NODE_PORT
  
  ```

  - 使用 `helm show values` 可以查看 chart 中的可配置选项

    > 如果使用的是本地chart，命令后跟的是chart的本地文件夹路径。

  ```shell
  [root@root ~]#  helm show values ./helm/nginx
  # Default values for nginx.
  # This is a YAML-formatted file.
  # Declare variables to be passed into your templates.
  
  replicaCount: 1
  
  image:
    repository: nginx
    pullPolicy: IfNotPresent
    # Overrides the image tag whose default is the chart appVersion.
    tag: ""
  
  imagePullSecrets: []
  nameOverride: ""
  fullnameOverride: ""
  
  serviceAccount:
    # Specifies whether a service account should be created
    create: true
    # Annotations to add to the service account
    annotations: {}
    # The name of the service account to use.
    # If not set and create is true, a name is generated using the fullname template
    name: ""
    ...
  ```

  然后，你可以使用 YAML 格式的文件覆盖上述任意配置项，并在安装过程中使用该文件

  - 安装过程中有两种方式传递配置数据：
    - `--values` (或 `-f`)：使用 YAML 文件覆盖配置。可以指定多次，优先使用最右边的文件。
    - `--set`：通过命令行的方式对指定项进行覆盖。
  - `helm install` 命令可以从多个来源进行安装：
    - chart 的仓库（如上所述）
    - 本地 chart 压缩包（`helm install nginx nginx-0.1.1.tgz`）
    - 解压后的 chart 目录（`helm install nginx ~/heml/nginx`）
    - 完整的 URL（`helm install nginx https://example.com/charts/nginx -1.2.3.tgz`）

- 升级release和失败时恢复

  当你想升级到 chart 的新版本，或是修改 release 的配置，你可以使用 `helm upgrade` 命令。

  ```shell
  [root@root ~]# helm upgrade nginx ./nginx
  ```

  也可以通过指定yaml文件来升级`helm uograde -f values.yaml nginx ./nginx`

  我们可以使用 `helm get values` 命令来看看配置值是否真的生效了

  ```shell
  [root@root ~]# helm get values nginx
  ```

  如在一次发布过程中，发生了不符合预期的事情，也很容易通过 `helm rollback [RELEASE] [REVISION]` 命令回滚到之前的发布版本。

  ```shell
  [root@root ~]# helm rollback nginx 1
  ```

  release 版本其实是一个增量修订（revision）。 每当发生了一次安装、升级或回滚操作，revision 的值就会加1。第一次 revision 的值永远是1。我们可以使用 `helm history [RELEASE]` 命令来查看一个特定 release 的修订版本号。

  ```shell
  [root@root ~]#  helm history nginx
  REVISION        UPDATED                         STATUS          CHART           APP VERSION     DESCRIPTION     
  1               Wed Aug 11 10:23:14 2021        superseded      nginx-0.1.0     1.16.0          Install complete
  2               Wed Aug 11 10:50:00 2021        deployed        nginx-0.1.0     1.16.0          Upgrade complete
  ```

- 查询应用列表

  使用`helm list`查询本地helm应用列表

  ```shell
  [root@root ~]# helm list
  NAME    NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
  nginx   default         2               2021-08-11 10:50:00.817310293 +0800 CST deployed        nginx-0.1.0     1.16.0 
  ```

- 卸载应用

  使用 `helm uninstall` 命令从集群中卸载一个 release：

  ```shell
  [root@root ~]# helm uninstall nginx
  release "nginx" uninstalled
  ```

  在上一个 Helm 版本中，当一个 release 被删除，会保留一条删除记录。而在 Helm 3 中，删除也会移除 release 的记录。 如果你想保留删除记录，使用 `helm uninstall --keep-history`。使用 `helm list --uninstalled` 只会展示使用了 `--keep-history` 删除的 release。

  
