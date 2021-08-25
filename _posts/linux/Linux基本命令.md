1. 防火墙

   1. 查看防火墙状态

      ```shell
      systemctl status firewalld
      ```

   2. 关闭防火墙

      ```shell
      service firewalld stop
      ```

      

2. 修改yum源

    [CentOS7-Base-163.repo](C:\Users\Lenovo\Desktop\yum\CentOS7-Base-163.repo) 

    [docker-ce.repo](C:\Users\Lenovo\Desktop\yum\docker-ce.repo) 

    [k8s.repo](C:\Users\Lenovo\Desktop\yum\k8s.repo) 

    [Centos-7.repo](C:\Users\Lenovo\Desktop\yum\Centos-7.repo) 

```
yum clean all  
yum makecache
```

3. vim 全局修改

   > :%s/from/to/g

   

