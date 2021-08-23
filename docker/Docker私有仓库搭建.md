## Registry仓库搭建

Docker官方提供一个搭建私有仓库的镜像Registry，只需要拉取镜像并启动，暴露5000端口即可使用

### 安装

```shell
# 检索镜像
[root@ahern ~]#  docker search registry
NAME                                 DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
registry                             The Docker Registry 2.0 implementation for s…   3225                [OK]                
distribution/registry                WARNING: NOT the registry official image!!! …   57                                      [OK]
stefanscherer/registry-windows       Containerized docker registry for Windows Se…   32                                      
budry/registry-arm                   Docker registry build for Raspberry PI 2 and…   18                                      
jc21/registry-ui                     A nice web interface for managing your Docke…   17                                      
deis/registry                        Docker image registry for the Deis open sour…   12                                      
anoxis/registry-cli                  You can list and delete tags from your priva…   11                                      [OK]
sixeyed/registry                     Docker Registry 2.6.0 running on Windows - N…   10                                      
pallet/registry-swift                Add swift storage support to the official do…   4                                       [OK]
allingeek/registry                   A specialization of registry:2 configured fo…   4                                       [OK]
arm32v6/registry                     The Docker Registry 2.0 implementation for s…   3                                       
goharbor/registry-photon                                                             3                                       
ibmcom/registry                      Docker Image for IBM Cloud private-CE (Commu…   1                                       
conjurinc/registry-oauth-server      Docker registry authn/authz server backed by…   1                                       
concourse/registry-image-resource                                                    1                                       
metadata/registry                    Metadata Registry is a tool which helps you …   1                                       [OK]
webhippie/registry                   Docker images for Registry                      1                                       [OK]
upmcenterprises/registry-creds                                                       1                                       
gisjedi/registry-proxy               Reverse proxy of registry mirror image gisje…   0                                       
kontena/registry                     Kontena Registry                                0                                       
dwpdigital/registry-image-resource   Concourse resource type                         0                                       
lorieri/registry-ceph                Ceph Rados Gateway (and any other S3 compati…   0                                       
convox/registry                                                                      0                                       
pivnet/registry-gcloud-image                                                         0                                       
nlgntr/registry-image-resource       Fork of https://github.com/concourse/registr…   0 

# 拉取镜像
[root@ahern ~]#  docker pull registry
Using default tag: latest
latest: Pulling from library/registry
0a6724ff3fcd: Pull complete 
d550a247d74f: Pull complete 
1a938458ca36: Pull complete 
acd758c36fc9: Pull complete 
9af6d68b484a: Pull complete 
Digest: sha256:d5459fcb27aecc752520df4b492b08358a1912fcdfa454f7d2101d4b09991daa
Status: Downloaded newer image for registry:latest

# 启动镜像
[root@ahern ~]#  docker run -d -v /root/registry:/var/lib/registry -p 5000:5000 --name registry registry:latest
fca6f0d4958e095b7828eb27a3ac8985b70d273968cfe3090b545ecc68b190e6
```

> Registry服务默认会将上传的镜像保存在容器的/var/lib/registry，我们将主机的/root/registry目录挂载到该目录,即可持久保存镜像到主机。
>
> 访问http://xxx.xxx.xxx.xxx:5000/v2查看服务是否正常

### 验证

