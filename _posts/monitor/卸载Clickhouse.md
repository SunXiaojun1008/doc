```shell
# 卸载及删除安装文件（需root权限）
yum list installed | grep clickhouse
 
yum remove -y clickhouse-common-static
 
yum remove -y clickhouse-server-common
 
rm -rf /var/lib/clickhouse
 
rm -rf /etc/clickhouse-*
 
rm -rf /var/log/clickhouse-server

# 删除rpm包的时候不调用卸载脚本
sudo rpm -e clickhouse-server.x86_64 --noscripts
```





```shell
export LATEST_VERSION=v21.7.8.58-stable
curl -O https://repo.clickhouse.tech/tgz/clickhouse-common-static-$LATEST_VERSION.tgz
curl -O https://repo.clickhouse.tech/tgz/clickhouse-common-static-dbg-$LATEST_VERSION.tgz
curl -O https://repo.clickhouse.tech/tgz/clickhouse-server-$LATEST_VERSION.tgz
curl -O https://repo.clickhouse.tech/tgz/clickhouse-client-$LATEST_VERSION.tgz

tar -zxvf clickhouse-common-static-21.7.8.58.tgz
sudo clickhouse-common-static-21.7.8.58/install/doinst.sh

tar -zxvf clickhouse-common-static-dbg-21.7.8.58.tgz
sudo clickhouse-common-static-dbg-21.7.8.58/install/doinst.sh

tar -zxvf clickhouse-server-21.7.8.58.tgz
sudo clickhouse-server-21.7.8.58/install/doinst.sh
sudo /etc/init.d/clickhouse-server start

tar -zxvf clickhouse-client-21.7.8.58.tgz
sudo clickhouse-client-21.7.8.58/install/doinst.sh

# 执行完上述脚本后，需要修改一下文件的拥有者，否则你会发现，当你重新启动服务时会一直失败，提示无权限
[root@dpm02 ~/clickhouse]#  chown -R clickhouse:clickhouse /usr/lib/debug/.build-id/
[root@dpm02 ~/clickhouse]#  chown -R clickhouse:clickhouse /etc/clickhouse-server/



```
