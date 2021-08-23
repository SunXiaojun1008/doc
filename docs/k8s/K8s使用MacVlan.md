准备CNI二进制文件

> 下载地址：https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz 

将下载的binary放置到每个node的/opt/cni/bin/目录

为kubelet配置CNI

在文件/etc/kubernetes/kubelet添加如下配置：

KUBELET_ARGS="--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"

macvlan CNI插件的配置文件/etc/cni/net.d/10-maclannet.conf如下：

```shell
{
    "name": "macvlannet",
    "type": "macvlan",
    "master": "ens33",
    "mode": "vepa"
    "isGateway": true,
    "ipMasq": false,
    "ipam": {
        "type": "host-local",
        "subnet": "192.168.56.0/24",
        "rangeStart": "192.168.56.110",
        "rangeEnd": "192.168.56.129",
        "gateway": "192.168.56.254",
        "routes": [
            { "dst": "0.0.0.0/0","gw":"192.168.56.254" }
        ]
    }
```

```shell
cat <<EOF | kubectl create -f -
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: macvlannet
EOF
```

