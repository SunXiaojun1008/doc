1. 下载镜像

   ```shell
   # 查询镜像
   docker search mysql
   # 拉取镜像
   docker pull mysql
   ```

   

2. 启动服务

   1. -v /root/mysql:/var/lib/mysql： 将容器内部的/var/lib/mysql挂载到宿主机的 /root/mysql目录
   2. --name mysql ：指定镜像名称为mysql
   3. -p 3306:3306：指定端口映射
   4. -e MYSQL_ROOT_PASSWORD=root： 指定root账户的密码
   5. -d : 以守护进程执行

   ```shell
   # 启动服务
   docker run --name mysql -v /root/mysql/data:/var/lib/mysql -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:latest
   ```

   

3. 查看服务

   ```
   docker ps | grep mysql
   ```

   