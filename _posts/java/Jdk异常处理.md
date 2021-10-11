### /root/presto/jdk1.8.0_301/bin/java: /lib/ld-linux.so.2: bad ELF interpreter: 没有那个文件或目录

- 异常信息:

jdk1.8解压安装后使用java -version测试是否配置成功，提示信息：

```shell
[root@dpm02 ~/presto]#  java -version
-bash: /root/presto/jdk1.8.0_301/bin/java: /lib/ld-linux.so.2: bad ELF interpreter: 没有那个文件或目录
```

- 解决方案

```shell
sudo yum install glibc.i686
```

