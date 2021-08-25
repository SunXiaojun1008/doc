1. 下载镜像

   ```shell
   docker search redis
   
   docker pull redis
   ```

2. 启动服务

   1. -p 6379:6379 : 将容器的6379端口映射到主机的6379端口
   2. -v $PWD/redis/data:/data : 将主机中当前目录下的data挂载到容器的/data
   3. redis-server --appendonly yes : 在容器执行redis-server启动命令，并打开redis持久化配置

   ```shell
   docker run -p 6379:6379 -v $PWD/redis/data:/data  -d redis:latest redis-server --appendonly yes
   ```

3. 查看结果

   ```shell
   docker ps | grep redis
   ```

   



