## 依赖

- crontab

- sshpass

  ```shell
  # yum install sshpass
  ```

- rsync

  ```shell
  # yum install rsync
  ```

## 脚本

- 定时从服务器A同步配置到服务器B

  - 在服务器A上输入`crontab -e`执行，打开crontab编辑器

    ```shell
    # crontab -e
    ```

  - 输入下列指令

    > */1 * * * * ：cron表达式，表示每分钟执行一次
    >
    > sshpass -p 'R00t@123'：免密

    ```shell
    */1 * * * * sshpass -p 'R00t@123' rsync -av -r --progress --exclude='*.properties' --exclude='jvm.config' /root/presto/presto-server-0.263/etc/* root@10.154.8.78:/root/presto/presto-server-0.263/etc/
    ```

- 定时从服务器A拉取配置到服务器B

  - 在服务器B上输入`crontab -e`执行，打开crontab编辑器

    ```shell
    # crontab -e
    ```

  - 输入下列指令

    ```shell
    */1 * * * * sshpass -p 'R00t@123' rsync -av -r --progress --exclude='*.properties' --exclude='jvm.config'  root@10.154.8.78:/root/presto/presto-server-0.263/etc/* /root/presto/presto-server-0.263/etc/
    ```
  
- 查看crontab日志

  ```shell
  # tail -f /var/log/cron
  ```

  
