#### Kubelet服务启动异常，Node cannot join

##### 查找原因:

> 10.154.5.55:6443: connect: connection refused
> Mar 12 09:33:42 ops0008 kubelet[468453]: E0312 09:33:42.338085  468453 reflector.go:123] object-"default"/"ops-job-squirtle-config-secret": Failed to list *v1.Secret: Get https://10.154.5.55:6443/api/v1/namespaces/default/secrets?fieldSelector=metadata.name%3Dops-job-squirtle-config-secret&limit=500&resourceVersion=0: dial tcp 10.154.5.55:6443: connect: connection refused
> Mar 12 09:33:42 ops0008 kubelet[468453]: E0312 09:33:42.537905  468453 reflector.go:123] object-"kube-system"/"coredns-token-rkf5q": Failed to list *v1.Secret: Get https://10.154.5.55:6443/api/v1/namespaces/kube-system/secrets?fieldSelector=metadata.name%3Dcoredns-token-rkf5q&limit=500&resourceVersion=0: dial tcp 10.154.5.55:6443: connect: connection refused
> Mar 12 09:33:42 ops0008 kubelet[468453]: E0312 09:33:42.574294  468453 reflector.go:280] k8s.io/kubernetes/pkg/kubelet/config/apiserver.go:46: Failed to watch *v1.Pod: Get https://10.154.5.55:6443/api/v1/pods?allowWatchBookmarks=true&fieldSelector=spec.nodeName%3Dops0008&resourceVersion=8953123&timeoutSeconds=553&watch=true: dial tcp 10.154.5.55:6443: connect: connection refused
> Mar 12 09:33:42 ops0008 kubelet[468453]: W0312 09:33:42.590826  468453 kubelet_pods.go:849] Unable to retrieve pull secret default/harbor-secret for default/ops-system-auth-7c8dbf4984-cw9kc due to failed to sync secret cache: timed out waiting for the condition.  The image pull may not succeed.
> Mar 12 09:33:42 ops0008 kubelet[468453]: I0312 09:33:42.590869  468453 kuberuntime_manager.go:439] Sandbox for pod "ops-system-auth-7c8dbf4984-cw9kc_default(cb92da54-e8b1-4c40-b9e4-a09e03d6f98f)" has no IP address.  Need to start a new one
> Mar 12 09:33:42 ops0008 kubelet[468453]: W0312 09:33:42.595111  468453 cni.go:328] <font color="red" weight="stroge">CNI failed to retrieve network namespace path: cannot find network namespace for the terminated container "202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a"</font>
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.727 [INFO][486068] plugin.go 442: Extracted identifiers ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" Node="ops0008" Orchestrator="k8s" WorkloadEndpoint="ops0008-k8s-ops--system--auth--7c8dbf4984--cw9kc-eth0"
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.731 [WARNING][486068] k8s.go 447: WorkloadEndpoint does not exist in the datastore, moving forward with the clean up ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" WorkloadEndpoint="ops0008-k8s-ops--system--auth--7c8dbf4984--cw9kc-eth0"
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.731 [INFO][486068] k8s.go 477: Cleaning up netns ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a"
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.731 [INFO][486068] k8s.go 484: Releasing IP address(es) ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a"
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.731 [INFO][486068] utils.go 168: Calico CNI releasing IP address ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a"
> Mar 12 09:33:42 ops0008 kubelet[468453]: E0312 09:33:42.741049  468453 reflector.go:123] object-"default"/"nsre-front-stitch-nginx-conf": Failed to list *v1.ConfigMap: Get https://10.154.5.55:6443/api/v1/namespaces/default/configmaps?fieldSelector=metadata.name%3Dnsre-front-stitch-nginx-conf&limit=500&resourceVersion=0: dial tcp 10.154.5.55:6443: connect: connection refused
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.776 [INFO][486086] ipam_plugin.go 299: Releasing address using handleID ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" HandleID="k8s-pod-network.202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" Workload="ops0008-k8s-ops--system--auth--7c8dbf4984--cw9kc-eth0"
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.777 [INFO][486086] ipam.go 1166: Releasing all IPs with handle 'k8s-pod-network.202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a'
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.780 [WARNING][486086] ipam_plugin.go 306: Asked to release address but it doesn't exist. Ignoring ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" HandleID="k8s-pod-network.202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" Workload="ops0008-k8s-ops--system--auth--7c8dbf4984--cw9kc-eth0"
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.780 [INFO][486086] ipam_plugin.go 317: Releasing address using workloadID ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" HandleID="k8s-pod-network.202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a" Workload="ops0008-k8s-ops--system--auth--7c8dbf4984--cw9kc-eth0"
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.780 [INFO][486086] ipam.go 1166: Releasing all IPs with handle 'default.ops-system-auth-7c8dbf4984-cw9kc'
> Mar 12 09:33:42 ops0008 kubelet[468453]: 2021-03-12 09:33:42.784 [INFO][486068] k8s.go 490: Teardown processing complete. ContainerID="202801ed5cb38ad079973727cdb083cb9b700cffb9ba75a2cf5ce5c5221aff3a"
> Mar 12 09:33:42 ops0008 kubelet[468453]: E0312 09:33:42.937676  468453 reflector.go:123] object-"default"/"harbor-secret": Failed to list *v1.Secret: Get https://10.154.5.55:6443/api/v1/namespaces/default/secrets?fieldSelector=metadata.name%3Dharbor-secret&limit=500&resourceVersion=0: dial tcp 10.154.5.55:6443: connect: 

##### 解决方案：

After experiencing the same issue, editing /var/lib/kubelet/config.yaml to add:

修改k8s的配置文件config.yaml(默认路径为：`/var/lib/kubelet/config.yaml`)，增加如下配置

```yaml
featureGates:
  CSIMigration: false
```



#### 批量强制删除异常Pod

kubectl get po -A -o wide | grep Terminating | awk ‘{print $2}’ | xargs kubectl  delete po --grace-period=0 --force

