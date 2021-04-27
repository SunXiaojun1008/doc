## 基础环境

### 配置镜像拉取

1. 设置host

   ```shell
   vim /etc/hosts
   
   10.154.4.104 registry.paas
   ```

   

2. 编辑/etc/docker/daemon.json 文件

   ```shell
   {"insecure-registries": ["registry.paas"]}
   ```

   

3. 重启docker服务

   ```shell
   systemctl restart docker.service
   ```

4. 登录docker

   ```shell
   docker login registry.paas -u admin -p 8cDcos11
   ```
   
5. 创建secret

   ```
   kubectl create secret docker-registry docker-image-secret --namespace=thanos --docker-username=admin --docker-password=8cDcos11 --docker-server=registry.paas 
   
   ```

   

### 创建Namespace

1. 创建Namespace

   > 在K8S集群中创建一个名为thanos的命名空间,使用yaml文件：`thanos-namespaces.yaml`，具体内容如下:

   ```yaml
   apiVersion: v1
   kind: Namespace
   metadata:
     name: thanos
   ```

### 创建StorageClass

1. 创建ServiceAccount

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: nfs-provisioner
     namespace: thanos
   ```

2. 创建ClusterRole，并指定权限

   ```yaml
   kind: ClusterRole
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: nfs-provisioner-runner
     namespace: thanos
   rules:
     - apiGroups: [""]
       resources: ["persistentvolumes"]
       verbs: ["get", "list", "watch", "create", "delete"]
     - apiGroups: [""]
       resources: ["persistentvolumeclaims"]
       verbs: ["get", "list", "watch", "update"]
     - apiGroups: ["storage.k8s.io"]
       resources: ["storageclasses"]
       verbs: ["get", "list", "watch"]
     - apiGroups: [""]
       resources: ["events"]
       verbs: ["create", "update", "patch"]
     - apiGroups: [""]
       resources: ["services", "endpoints"]
       verbs: ["get"]
     - apiGroups: ["extensions"]
       resources: ["podsecuritypolicies"]
       resourceNames: ["nfs-provisioner"]
       verbs: ["use"]
   ```

3. 创建ClusterRoleBinding

   ```yaml
   kind: ClusterRoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: run-nfs-provisioner
     namespace: thanos
   subjects:
     - kind: ServiceAccount
       name: nfs-provisioner
        # replace with namespace where provisioner is deployed
       namespace: thanos
   roleRef:
     kind: ClusterRole
     name: nfs-provisioner-runner
     apiGroup: rbac.authorization.k8s.io
   ```

4. 创建Role

   ```yaml
   kind: Role
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-provisioner
     namespace: thanos
   rules:
     - apiGroups: [""]
       resources: ["endpoints"]
       verbs: ["get", "list", "watch", "create", "update", "patch"]
   ```

5. 创建RoleBinding

   ```yaml
   kind: RoleBinding
   apiVersion: rbac.authorization.k8s.io/v1
   metadata:
     name: leader-locking-nfs-provisioner
     namespace: thanos
   subjects:
     - kind: ServiceAccount
       name: nfs-provisioner
       # replace with namespace where provisioner is deployed
       namespace: thanos
   roleRef:
     kind: Role
     name: leader-locking-nfs-provisioner
     apiGroup: rbac.authorization.k8s.io
   
   ```

6. 创建Deployment

   ```yaml
   kind: Deployment
   apiVersion: apps/v1
   metadata:
     name: nfs-provisioner
     namespace: thanos
     labels:
       app: nfs-provisioner
   spec:
     replicas: 1
     strategy:
       type: Recreate
     selector:
       matchLabels:
         app: nfs-provisioner
     template:
       metadata:
         labels:
           app: nfs-provisioner
       spec:
         serviceAccount: nfs-provisioner
         containers:
           - name: nfs-provisioner
             image: registry.cn-hangzhou.aliyuncs.com/open-ali/nfs-client-provisioner
             volumeMounts:
               - name: nfs-client-root
                 mountPath: /persistentvolumes
             env:
               - name: PROVISIONER_NAME
                 value: example.com/nfs
               - name: NFS_SERVER
                 value: 10.154.8.173  
               - name: NFS_PATH
                 value: /nfs/data
         volumes:
           - name: nfs-client-root
             nfs:
               server: 10.154.8.173
               path: /nfs/data
   ```

7. 创建StorageClass

   ```yaml
   kind: StorageClass
   apiVersion: storage.k8s.io/v1
   metadata:
     name: example-nfs
     namespace: thanos
   provisioner: example.com/nfs
   ```

### 创建Prometheus-Node

1. 创建Service

   ```yaml
   kind: Service
   apiVersion: v1
   metadata:
     name: prometheus-node-headless
     namespace: thanos
     labels:
       app.kubernetes.io/name: prometheus-node
   spec:
     type: NodePort
   #  clusterIP: None
     selector:
       app.kubernetes.io/name: prometheus-node
     ports:
     - name: web
       protocol: TCP
       port: 9090
       targetPort: web
       nodePort: 30212
   ```

   

2. 创建一个名为prometheus-node的账号

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: prometheus-node
     namespace: thanos
   ```

3. 创建一个名为：prometheus-node的ClusterRole，拥有以下权限。若服务无需访问本地服务，可以考虑不进行创建

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRole
   metadata:
     name: prometheus-node
     namespace: thanos
   rules:
   - apiGroups: [""]
     resources:
     - nodes
     - nodes/proxy
     - nodes/metrics
     - services
     - endpoints
     - pods
     verbs: ["get", "list", "watch"]
   - apiGroups: [""]
     resources: ["configmaps"]
     verbs: ["get"]
   - nonResourceURLs: ["/metrics"]
     verbs: ["get"]
   ```

4. 创建ServiceAccount和ClusterRole的绑定关系

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRoleBinding
   metadata:
     name: prometheus-node
   subjects:
     - kind: ServiceAccount
       name: prometheus-node
       namespace: thanos
   roleRef:
     kind: ClusterRole
     name: prometheus-node
     apiGroup: rbac.authorization.k8s.io
   ```

5. StatefulSet

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: prometheus-node
     namespace: thanos # 变量，需替换
     labels:
       app.kubernetes.io/name: prometheus-node
   spec:
     serviceName: prometheus-node-headless
     podManagementPolicy: Parallel
     replicas: 1
     selector:
       matchLabels:
         app.kubernetes.io/name: prometheus-node
     template:
       metadata:
         labels:
           app.kubernetes.io/name: prometheus-node
       spec:
         serviceAccountName: prometheus-node
         securityContext:
           fsGroup: 2000
           runAsNonRoot: true
           runAsUser: 1000
         affinity:
           podAntiAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
             - labelSelector:
                 matchExpressions:
                 - key: app.kubernetes.io/name
                   operator: In
                   values:
                   - prometheus-node
               topologyKey: kubernetes.io/hostname
         containers:
         - name: prometheus-node
           image: quay.io/prometheus/prometheus:v2.15.2
           args:
           - --config.file=/var/config/prometheus/node/$(MY_POD_NAMESPACE)/$(MY_POD_NAME)/$(MY_POD_NAME).yaml # 变量，需替换
           - --storage.tsdb.path=/prometheus
           - --storage.tsdb.retention.time=1d
           - --web.route-prefix=/
           - --web.enable-lifecycle
           - --storage.tsdb.no-lockfile
           - --storage.tsdb.min-block-duration=2h
           - --storage.tsdb.max-block-duration=2h
           - --log.level=debug
           env:
             - name: MY_POD_NAME
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.name
             - name: MY_POD_NAMESPACE
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.namespace
           ports:
           - containerPort: 9090
             name: web
             protocol: TCP
           livenessProbe:
             failureThreshold: 6
             httpGet:
               path: /-/healthy
               port: web
               scheme: HTTP
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 3
           readinessProbe:
             failureThreshold: 120
             httpGet:
               path: /-/ready
               port: web
               scheme: HTTP
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 3
           volumeMounts:
           - mountPath: /var/config/prometheus/node
             name: prometheus-config
           - mountPath: /prometheus
             name: prometheus-node-storage
         volumes:
         - name: prometheus-config
           nfs:
             server: 10.154.8.173
             path: /nfs/data/dpm-node/prometheus/node
     volumeClaimTemplates:
     - metadata:
         name: prometheus-node-storage
         labels:
           app.kubernetes.io/name: prometheus-node
       spec:
         accessModes:
         - ReadWriteOnce
         resources:
           requests:
             storage: 5Gi
         volumeMode: Filesystem
         storageClassName: example-nfs
   ```

   

### 创建Prometheus-Server

1. 创建Service

   ```yaml
   kind: Service
   apiVersion: v1
   metadata:
     name: prometheus-headless
     namespace: thanos
     labels:
       app.kubernetes.io/name: prometheus
   spec:
     type: ClusterIP
     clusterIP: None
     selector:
       app.kubernetes.io/name: prometheus
     ports:
     - name: web
       protocol: TCP
       port: 9090
       targetPort: web
   #    nodePort: 30211
     - name: grpc
       port: 10901
       targetPort: grpc
   ```

2. 创建ServiceAccount

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: prometheus
     namespace: thanos
   ```

3. 创建ClusterRole，同时定义操作权限

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRole
   metadata:
     name: prometheus
     namespace: thanos
   rules:
   - apiGroups: [""]
     resources:
     - nodes
     - nodes/proxy
     - nodes/metrics
     - services
     - endpoints
     - pods
     verbs: ["get", "list", "watch"]
   - apiGroups: [""]
     resources: ["configmaps"]
     verbs: ["get"]
   - nonResourceURLs: ["/metrics"]
     verbs: ["get"]
   ```

4. 创建ClusterRoleBinding,将ServiceAccount与ClusterRole进行绑定

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRoleBinding
   metadata:
     name: prometheus
   subjects:
     - kind: ServiceAccount
       name: prometheus
       namespace: thanos
   roleRef:
     kind: ClusterRole
     name: prometheus
     apiGroup: rbac.authorization.k8s.io
   ```

5. 创建configmap

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: prometheus-config-tmpl
     namespace: thanos
   data:
     prometheus.yaml.tmpl: |-
       global:
         scrape_interval: 5s
         evaluation_interval: 5s
         external_labels:
           cluster: prometheus-ha
           prometheus_replica: $(POD_NAME)
       rule_files:
       - /etc/prometheus/rule/*.yaml
       remote_write:
       - url: http://prometheus-pulsar-adaptor:8080/adapter/pulsar/produce
       scrape_configs:
       - job_name: 'federate'
         scrape_interval: 15s
         honor_labels: true
         metrics_path: '/federate'
         params:
           'match[]':
           - '{job="prometheus"}'
           - '{job=~"instance.*"}'
         kubernetes_sd_configs:
         - role: pod
         relabel_configs:
         - source_labels: [__meta_kubernetes_namespace]
           action: keep 
           target_label: thanos
         - source_labels: [__meta_kubernetes_pod_name]
           action: keep
           regex: prometheus-node-.*
           target_label: __meta_kubernetes_pod_name
          
   ```

   

6. 创建服务Pod（StatefulSet），包含两个镜像Prometheus&Sidecar

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: prometheus
     namespace: thanos
     labels:
       app.kubernetes.io/name: thanos-query
   spec:
     serviceName: prometheus-headless
     podManagementPolicy: Parallel
     replicas: 1
     selector:
       matchLabels:
         app.kubernetes.io/name: prometheus
     template:
       metadata:
         labels:
           app.kubernetes.io/name: prometheus
       spec:
         serviceAccountName: prometheus
         securityContext:
           fsGroup: 2000
           runAsNonRoot: true
           runAsUser: 1000
         affinity:
           podAntiAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
             - labelSelector:
                 matchExpressions:
                 - key: app.kubernetes.io/name
                   operator: In
                   values:
                   - prometheus
               topologyKey: kubernetes.io/hostname
         containers:
         - name: prometheus
           image: quay.io/prometheus/prometheus:v2.15.2
           args:
           - --config.file=/etc/prometheus/config_out/prometheus.yaml
           - --storage.tsdb.path=/prometheus
           - --storage.tsdb.retention.time=1d
           - --web.route-prefix=/
           - --web.enable-lifecycle
           - --storage.tsdb.no-lockfile
           - --storage.tsdb.min-block-duration=2h
           - --storage.tsdb.max-block-duration=2h
           - --log.level=debug
           ports:
           - containerPort: 9090
             name: web
             protocol: TCP
           livenessProbe:
             failureThreshold: 6
             httpGet:
               path: /-/healthy
               port: web
               scheme: HTTP
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 3
           readinessProbe:
             failureThreshold: 120
             httpGet:
               path: /-/ready
               port: web
               scheme: HTTP
             periodSeconds: 5
             successThreshold: 1
             timeoutSeconds: 3
           volumeMounts:
           - mountPath: /etc/prometheus/config_out
             name: prometheus-config-out
           - mountPath: /prometheus
             name: prometheus-storage
           - mountPath: /etc/prometheus/rule
             name: dpm-node-config
         - name: thanos
           image: quay.io/thanos/thanos:v0.11.0
           args:
           - sidecar
           - --log.level=debug
           - --tsdb.path=/prometheus
           - --prometheus.url=http://127.0.0.1:9090
   #        - --objstore.config-file=/etc/thanos/objectstorage.yaml
           - --reloader.config-file=/etc/prometheus/config/prometheus.yaml.tmpl # 实现configmap的热更新
           - --reloader.config-envsubst-file=/etc/prometheus/config_out/prometheus.yaml
           env:
           - name: POD_NAME
             valueFrom:
               fieldRef:
                 fieldPath: metadata.name
           ports:
           - name: http-sidecar
             containerPort: 10902
           - name: grpc
             containerPort: 10901
           livenessProbe:
               httpGet:
                 port: 10902
                 path: /-/healthy
           readinessProbe:
             httpGet:
               port: 10902
               path: /-/ready
           volumeMounts:
           - name: prometheus-config-tmpl
             mountPath: /etc/prometheus/config
           - name: prometheus-config-out
             mountPath: /etc/prometheus/config_out
           - name: prometheus-storage
             mountPath: /prometheus
   #        - name: thanos-objectstorage
   #          subPath: objectstorage.yaml
   #          mountPath: /etc/thanos/objectstorage.yaml
         volumes:
         - name: prometheus-config-tmpl
           configMap:
             defaultMode: 420 #secrets与configmap 默认readonly
             name: prometheus-config-tmpl
         - name: dpm-node-config
           nfs:
             server: 10.154.8.173
             path: /nfs/data/dpm-node/prometheus/master
         - name: prometheus-config-out
           emptyDir: {}
     volumeClaimTemplates:
     - metadata:
         name: prometheus-storage
         labels:
           app.kubernetes.io/name: prometheus
       spec:
         accessModes:
         - ReadWriteOnce
         resources:
           requests:
             storage: 5Gi
         volumeMode: Filesystem
         storageClassName: example-nfs 
   ```

### 创建thanos-query

1. 创建Service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: thanos-query
     namespace: thanos
     labels:
       app.kubernetes.io/name: thanos-query
   spec:
     ports:
     - name: grpc
       port: 10901
       targetPort: grpc
     - name: http
       port: 9090
       targetPort: http
     selector:
       app.kubernetes.io/name: thanos-query
   ```

   

2. 创建Pod

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: thanos-query
     namespace: thanos
     labels:
       app.kubernetes.io/name: thanos-query
   spec:
     replicas: 1
     selector:
       matchLabels:
         app.kubernetes.io/name: thanos-query
     template:
       metadata:
         labels:
           app.kubernetes.io/name: thanos-query
       spec:
         affinity:
           podAntiAffinity:
             preferredDuringSchedulingIgnoredDuringExecution:
             - podAffinityTerm:
                 labelSelector:
                   matchExpressions:
                   - key: app.kubernetes.io/name
                     operator: In
                     values:
                     - thanos-query
                 topologyKey: kubernetes.io/hostname
               weight: 100
         containers:
         - args:
           - query
           - --log.level=debug
           - --query.auto-downsampling
           - --grpc-address=0.0.0.0:10901
           - --http-address=0.0.0.0:9090
           - --query.partial-response
           - --query.replica-label=prometheus_replica
           - --query.replica-label=rule_replica
           - --store=dnssrv+_grpc._tcp.prometheus-headless.thanos.svc.cluster.local
           - --store=dnssrv+_grpc._tcp.thanos-rule.thanos.svc.cluster.local
           - --store=dnssrv+_grpc._tcp.thanos-store.thanos.svc.cluster.local
           image: thanosio/thanos:v0.11.0
           livenessProbe:
             failureThreshold: 4
             httpGet:
               path: /-/healthy
               port: 9090
               scheme: HTTP
             periodSeconds: 30
           name: thanos-query
           ports:
           - containerPort: 10901
             name: grpc
           - containerPort: 9090
             name: http
           readinessProbe:
             failureThreshold: 20
             httpGet:
               path: /-/ready
               port: 9090
               scheme: HTTP
             periodSeconds: 5
           terminationMessagePolicy: FallbackToLogsOnError
         terminationGracePeriodSeconds: 120
   ```

### 创建thanos-ruler

1. 创建configmap

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: thanos-rules
     labels:
       name: thanos-rules
     namespace: thanos
   data:
     record.rules.yaml: |-
       groups:
       - name: k8s.rules
         rules:
         - expr: |
             sum(rate(container_cpu_usage_seconds_total{job="cadvisor", image!="", container!=""}[5m])) by (namespace)
           record: namespace:container_cpu_usage_seconds_total:sum_rate
         - expr: |
             sum(container_memory_usage_bytes{job="cadvisor", image!="", container!=""}) by (namespace)
           record: namespace:container_memory_usage_bytes:sum
         - expr: |
             sum by (namespace, pod, container) (
               rate(container_cpu_usage_seconds_total{job="cadvisor", image!="", container!=""}[5m])
             )
           record: namespace_pod_container:container_cpu_usage_seconds_total:sum_rate
   ```

2. 创建Service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     labels:
       app.kubernetes.io/name: thanos-rule
     name: thanos-rule
     namespace: thanos
   spec:
     clusterIP: None
     ports:
     - name: grpc
       port: 10901
       targetPort: grpc
     - name: http
       port: 10902
       targetPort: http
     selector:
       app.kubernetes.io/name: thanos-rule
   ```

3. 创建pod

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     labels:
       app.kubernetes.io/name: thanos-rule
     name: thanos-rule
     namespace: thanos
   spec:
     replicas: 2
     selector:
       matchLabels:
         app.kubernetes.io/name: thanos-rule
     serviceName: thanos-rule
     podManagementPolicy: Parallel
     template:
       metadata:
         labels:
           app.kubernetes.io/name: thanos-rule
       spec:
         containers:
         - args:
           - rule
           - --grpc-address=0.0.0.0:10901
           - --http-address=0.0.0.0:10902
           - --rule-file=/etc/thanos/rules/*rules.yaml
   #        - --objstore.config-file=/etc/thanos/objectstorage.yaml
           - --data-dir=/var/thanos/rule
           - --label=rule_replica="$(NAME)"
           - --alert.label-drop="rule_replica"
           - --query=dnssrv+_http._tcp.thanos-query.thanos.svc.cluster.local
           env:
           - name: NAME
             valueFrom:
               fieldRef:
                 fieldPath: metadata.name
           image: thanosio/thanos:v0.11.0
           livenessProbe:
             failureThreshold: 24
             httpGet:
               path: /-/healthy
               port: 10902
               scheme: HTTP
             periodSeconds: 5
           name: thanos-rule
           ports:
           - containerPort: 10901
             name: grpc
           - containerPort: 10902
             name: http
           readinessProbe:
             failureThreshold: 18
             httpGet:
               path: /-/ready
               port: 10902
               scheme: HTTP
             initialDelaySeconds: 10
             periodSeconds: 5
           terminationMessagePolicy: FallbackToLogsOnError
           volumeMounts:
           - mountPath: /var/thanos/rule
             name: data
             readOnly: false
    #       - name: thanos-objectstorage
    #         subPath: objectstorage.yaml
    #         mountPath: /etc/thanos/objectstorage.yaml
           - name: thanos-rules
             mountPath: /etc/thanos/rules
         volumes:
    #     - name: thanos-objectstorage
    #       secret:
    #         secretName: thanos-objectstorage
         - name: thanos-rules
           configMap:
             name: thanos-rules
     volumeClaimTemplates:
     - metadata:
         labels:
           app.kubernetes.io/name: thanos-rule
         name: data
       spec:
         accessModes:
         - ReadWriteOnce
         resources:
           requests:
             storage: 5Gi
         storageClassName: example-nfs 
   ```

   

### 创建thanos-compact

### 创建thanos-store-gateway

### 创建dpm-master

1. 创建Service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: dpm-master
     namespace: thanos
     labels:
       app.kubernetes.io/name: dpm-master
   spec:
     ports:
       - port: 8080
         targetPort: 8080
         protocol: TCP
         nodePort: 30302
         name: http-port
     type: NodePort
     selector:
       app.kubernetes.io/name: dpm-master
   ```

   

2. 创建Configmap

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: dpm-master-configmap
     namespace: thanos
     labels:
       name: dpm-master-configmap
     namespace: thanos
   data:
     application.properties: |-
       server.port=8080
       spring.application.name=dpm-master
       spring.http.encoding.charset=UTF-8
       spring.http.encoding.force=true
       spring.jackson.serialization.write-dates-as-timestamps=true
       spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
       spring.jackson.time-zone=Asia/Shanghai
       spring.profiles.active=${PROFILES_ACTIVE:local}
       spring.session.store-type=none
       spring.datasource.url=jdbc:postgresql://10.139.5.215:5432/hitch?characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Shanghai
       spring.datasource.username=ENC(BxR5s6Xi1tKrB2G9XSQ5xd/PXtNMZP1F)
       spring.datasource.password=ENC(26oqzzxAz+IWnZjXq50P2w==)
       spring.datasource.driver-class-name=org.postgresql.Driver
       spring.datasource.type=com.zaxxer.hikari.HikariDataSource
       spring.datasource.hikari.minimum-idle=5
       spring.datasource.hikari.idle-timeout=180000
       spring.datasource.hikari.maximum-pool-size=10
       spring.datasource.hikari.auto-commit=true
       spring.datasource.hikari.pool-name=MyHikariCP
       spring.datasource.hikari.max-lifetime=1800000
       spring.datasource.hikari.connection-timeout=30000
       spring.datasource.hikari.connection-test-query=SELECT 1
       spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
       spring.kafka.consumer.properties.isolation.level=read_committed
       spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
       spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
       spring.kafka.consumer.group-id=cnvas-group
       spring.kafka.bootstrap-servers=ops0016:6667,ops0017:6667,ops0018:6667
       swagger2.enabled=true
       mybatis.configuration.map-underscore-to-camel-case=true
       mybatis.mapper-locations=classpath:mapper/*Mapper.xml
       mybatis.type-aliases-package=com.chinamobile.cmss.aiops.dpm.master.entity
       management.endpoints.web.exposure.include=*
       management.endpoint.info.enabled=true
       management.endpoint.health.show-details=always
       jasypt.encryptor.password=Y252YXM
       pagehelper.reasonable=true
       dpm.kafka.topic=dpm-rule-listen
       dpm.kafka.dlt=dpm-rule-listen.DLT
       dpm.kafka.enabled=true
       dpm.kafka.kerberos.userPrincipal=kafka/ops0017@BCHKDC
       dpm.kafka.kerberos.keyTabPath=/var/config/kafka.service.keytab
       dpm.kafka.kerberos.enabled=true
       dpm.kafka.kerberos.server-name=kafka
       dpm.kafka.kerberos.krb5=/var/config/krb5.conf
   ```

3. 创建Pod

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: dpm-master
     namespace: thanos
     labels:
       app.kubernetes.io/name: dpm-master
   spec:
     replicas: 1
     strategy:
       type: Recreate
     selector:
       matchLabels:
         app.kubernetes.io/name: dpm-master
     template:
       metadata:
         labels:
           app.kubernetes.io/name: dpm-master
       spec:
         hostAliases:
         - ip: "10.154.5.210"
           hostnames:
           - "ops0014"
         - ip: "10.154.5.213"
           hostnames:
           - "ops0015"
         - ip: "10.154.5.207"
           hostnames:
           - "ops0016"
         - ip: "10.154.5.208"
           hostnames:
           - "ops0017"
         - ip: "10.154.5.209"
           hostnames:
           - "ops0018"
         containers:
         - name: dpm-master
           image: registry.paas/ops/dpm-master:2021-4
           imagePullPolicy: Always
           volumeMounts:
           - mountPath: /var/logs/master
             name: dpm-master-logs
           - mountPath: /var/config/application.properties
          name: dpm-master-properties
             subPath: application.properties
           - mountPath: /var/config/
             name: dpm-master-config
           ports:
           - containerPort: 8080
             protocol: TCP 
         restartPolicy: Always
         imagePullSecrets:
         - name: docker-image-secret
         volumes:
         - name: dpm-master-properties
           configMap:
             name: dpm-master-configmap
         - name: dpm-master-config
           nfs:
             server: 10.154.8.173
             path: /nfs/data/dpm-master
         - name: dpm-master-logs
           hostPath:
             path: /apps/logs/bk-ops/guard/bk-prometheus-master
   ```
   
   

### 创建dpm-node

1. 创建ServiceAccount

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: dpm-node
     namespace: thanos
   ```

2. 创建一个名为：dpm-node的ClusterRole，拥有以下权限

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRole
   metadata:
     name: dpm-node
     annotations:
       rbac.authorization.kubernetes.io/autoupdate: true
   rules:
   - apiGroups:
     - '*'
     resources:
     - '*'
     verbs:
     - '*'
   - nonResourceURLs:
     - '*'
     verbs:
     - '*'
   ```

   

3. 创建ServiceAccount和ClusterRole的绑定关系

   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: dpm-node
     annotations:
       rbac.authorization.kubernetes.io/autoupdate: true
   subjects:
   - kind: ServiceAccount
     name: dpm-node
     namespace: thanos
   roleRef:
     kind: ClusterRole
     name: dpm-node
     apiGroup: rbac.authorization.k8s.io
   ```

   

4. 创建Service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: dpm-node
     namespace: thanos
     labels:
       app.kubernetes.io/name: dpm-node
   spec:
     ports:
       - port: 8080
         targetPort: 8080
         protocol: TCP
         nodePort: 30301
         name: http-port
     type: NodePort
     selector:
       app.kubernetes.io/name: dpm-node
   ```

   

5. 创建configmap

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: dpm-node-configmap
     namespace: thanos
     labels:
       name: dpm-node-configmap
     namespace: thanos
   data:
     application.properties: |-
       server.port=8080
       spring.kafka.producer.value-serializer=org.springframework.kafka.support.serializer.JsonSerializer
       spring.kafka.consumer.properties.isolation.level=read_committed
       spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
       spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
       spring.kafka.consumer.group-id=dpm.${dpm.resourceId}
       spring.kafka.bootstrap-servers=ops0016:6667,ops0017:6667,ops0018:6667
       spring.redis.lettuce.pool.max-active=20
       spring.redis.lettuce.pool.max-wait=-1
       spring.redis.lettuce.pool.max-idle=8
       spring.redis.lettuce.pool.min-idle=8
       spring.redis.password=bc-ops-1qaz
       spring.redis.sentinel.master=mymaster
       spring.redis.sentinel.nodes=10.154.5.134:26379,10.154.5.167:26379
       spring.redis.timeout=10000
       spring.redis.database=12
       spring.application.name=dpm-node
       spring.http.encoding.charset=UTF-8
       spring.http.encoding.force=true
       spring.jackson.serialization.write-dates-as-timestamps=true
       spring.session.store-type=none
       swagger2.enabled=true
       management.endpoints.web.exposure.include=*
       management.endpoint.info.enabled=true
       management.endpoint.health.show-details=always
       dpm.init.enabled=false
       dpm.resourceId=123
       dpm.azId=22222
       dpm.master.url=http://dpm-master:8080
       dpm.redis.exclude=false
       dpm.redis.enabled=true
       dpm.prometheus.config.base-path=/var/config/prometheus
       dpm.prometheus.config.template=prometheus.yaml
       dpm.prometheus.config.max-count=500
       dpm.prometheus.config.server.namespace=dpm
       dpm.prometheus.config.server.scheme=http
       dpm.prometheus.config.server.label=app=app.kubernetes.io/name=bk-prometheus
       dpm.prometheus.config.server.port=9090
       dpm.prometheus.config.server.reload-path=/-/reload
       dpm.prometheus.config.items.business.namespace=default
       dpm.prometheus.config.items.business.scheme=http
       dpm.prometheus.config.items.business.label=app=app.kubernetes.io/name=bk-prometheus-node
       dpm.prometheus.config.items.business.port=9090
       dpm.prometheus.config.items.business.reload-path=/-/reload
       dpm.prometheus.config.items.management.namespace=dpm
       dpm.prometheus.config.items.management.scheme=http
       dpm.prometheus.config.items.management.label=app.kubernetes.io/name=bk-prometheus-node
       dpm.prometheus.config.items.management.port=9090
       dpm.prometheus.config.items.management.reload-path=/-/reload
       dpm.prometheus.config.items.default.namespace=dpm
       dpm.prometheus.config.items.default.scheme=http
       dpm.prometheus.config.items.default.label=app.kubernetes.io/name=bk-prometheus-node
       dpm.prometheus.config.items.default.port=9090
       dpm.prometheus.config.items.default.reload-path=/-/reload
       dpm.rule.sync-url=${dpm.master.url}/rule/full
       dpm.rule.redis-key=test4
       dpm.rule.alert-storage-url=/var/config/prometheus/master/alert.yaml
       dpm.rule.record-storage-url=/var/config/prometheus/master/record.yaml
       dpm.kafka.topic=dpm-rule-listen
       dpm.kafka.dlt=dpm-rule-listen.DLT
       dpm.kafka.enabled=true
       dpm.kafka.kerberos.userPrincipal=kafka/ops0017@BCHKDC
       dpm.kafka.kerberos.keyTabPath=/var/config/kafka.service.keytab
       dpm.kafka.kerberos.enabled=true
       dpm.kafka.kerberos.server-name=kafka
       dpm.kafka.kerberos.krb5=/var/config/krb5.conf
       dpm.pulsar.client.serviceUrl=pulsar://10.254.8.27:43170
       dpm.pulsar.producer.topic=persistent://dpm_user/dpm-app/performance
       dpm.pulsar.consumer.topic=persistent://dpm_user/dpm-app/performance
       dpm.pulsar.consumer.subcriptionName=dpm_group
       dpm.rocketmq.client.url=10.154.8.175:9876
       dpm.rocketmq.client.instanceName=dpm-mq
       dpm.rocketmq.producer.topic=dpm-storage
       dpm.rocketmq.producer.group=dpm-producer-group
       dpm.rocketmq.consumer.topic=dpm-storage
       dpm.rocketmq.consumer.group=dpm-consumer-group
       logging.level.root=INFO
   ```
   
6. 创建Pod

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: dpm-node
     namespace: thanos
     labels:
       app.kubernetes.io/name: dpm-node
   spec:
     replicas: 1
     strategy:
       type: Recreate
     selector:
       matchLabels:
         app.kubernetes.io/name: dpm-node
     template:
       metadata:
         labels:
           app.kubernetes.io/name: dpm-node
       spec:
         serviceAccountName: dpm-node
         hostAliases:
         - ip: "10.154.5.210"
           hostnames:
           - "ops0014"
         - ip: "10.154.5.213"
           hostnames:
           - "ops0015"
         - ip: "10.154.5.207"
           hostnames:
           - "ops0016"
         - ip: "10.154.5.208"
           hostnames:
           - "ops0017"
         - ip: "10.154.5.209"
           hostnames:
           - "ops0018"
         containers:
         - name: dpm-node
           image: registry.paas/ops/dpm-node:2021-31
           imagePullPolicy: Always
           volumeMounts:
           - mountPath: /var/logs/dpm
             name: dpm-node-logs
           - mountPath: /var/config/
             name: dpm-node-config
           - mountPath: /var/config/application.properties
             name: dpm-node-properties
             subPath: application.properties
           ports:
           - containerPort: 8080
             protocol: TCP 
         restartPolicy: Always
         imagePullSecrets:
         - name: docker-image-secret
         volumes:
         - name: dpm-node-properties
           configMap:
             name: dpm-node-configmap
         - name: dpm-node-config
           nfs:
             server: 10.154.8.173
             path: /nfs/data/dpm-node
         - name: dpm-node-logs
           hostPath:
             path: /apps/logs/bk-ops/guard/bk-prometheus-node
   ```



### 创建Promtheus-pulsar-adaptor

1. 创建Service

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: prometheus-pulsar-adaptor
     namespace: thanos
     labels:
       app.kubernetes.io/name: prometheus-pulsar-adaptor
   spec:
     ports:
       - port: 8080
         targetPort: 8080
         protocol: TCP
         nodePort: 30303
         name: http-port
     type: NodePort
     selector:
       app.kubernetes.io/name: prometheus-pulsar-adaptor
   ```

   

2. 创建Configmap

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: prometheus-pulsar-adaptor-configmap
     namespace: thanos
     labels:
       name: prometheus-pulsar-adaptor-configmap
   data:
     application.properties: |-
     	server.port=8080
       swagger2.enabled=true
       spring.application.name=prometheus-pulsar-adaptor
       management.endpoints.web.exposure.include=*
       management.endpoint.info.enabled=true
       management.endpoint.health.show-details=always
       dpm.pulsar.client.serviceUrl=pulsar://10.254.8.27:43170
       dpm.pulsar.client.token=eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJkZWZhdWx0X3JvbGUifQ.Y36gv3J2m0GtgLdvX1M85JSyeZD02QSOjuvoY5GcuSQ
       dpm.pulsar.client.listenerName=external
       dpm.pulsar.producer.topic=persistent://dpm_user/dpm-app/performance
       dpm.pulsar.consumer.topic=persistent://dpm_user/dpm-app/performance
       dpm.pulsar.consumer.subcriptionName=dpm_group
       logging.level.root=INFO
   
   ```

   

3. 创建Pod

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: prometheus-pulsar-adaptor
     namespace: thanos
     labels:
       app.kubernetes.io/name: prometheus-pulsar-adaptor
   spec:
     replicas: 1
     strategy:
       type: Recreate
     selector:
       matchLabels:
         app.kubernetes.io/name: prometheus-pulsar-adaptor
     template:
       metadata:
         labels:
           app.kubernetes.io/name: prometheus-pulsar-adaptor
       spec:
         containers:
         - name: prometheus-pulsar-adaptor
           image: registry.paas/ops/prometheus-pulsar-adaptor:2021-3
           imagePullPolicy: Always
           volumeMounts:
           - mountPath: /var/logs/dpm
             name: prometheus-pulsar-adaptor-logs
           - mountPath: /var/config/application.properties
             name: prometheus-pulsar-adaptor-properties
             subPath: application.properties
           ports:
           - containerPort: 8080
             protocol: TCP 
         restartPolicy: Always
         imagePullSecrets:
         - name: docker-image-secret
         volumes:
         - name: prometheus-pulsar-adaptor-properties
           configMap:
             name: prometheus-pulsar-adaptor-configmap
         - name: prometheus-pulsar-adaptor-logs
           hostPath:
             path: /apps/logs/bk-ops/guard/bk-prometheus-pulsar-adaptor
   ```

   

