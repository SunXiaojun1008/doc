### 介绍

promgen是一个基于Python开发的、方便prometheus 配置文件生成的工具，可以帮助我们生成以及管理prometheus的配置文件，同时可以配置案例alert 规则以及通知选项。

![img](https://img2020.cnblogs.com/blog/562987/202005/562987-20200521110627546-1295788907.png)

### 安装

从github下载安装包

```shell
wget https://codeload.github.com/line/promgen/zip/master
```

```shell
# Create promgen setting directory.
mkdir -p ~/.config/promgen
chmod 777 ~/.config/promgen

# Initialize required settings with Docker container
# This will prompt you for connection settings for your database and Redis broker
# using the standard DSN syntax.
# Database example: mysql://root:root@106.14.205.212:3306/promgen
# Broker example: redis://localhost:6379/0
docker run --rm -it -v ~/.config/promgen:/etc/promgen/ line/promgen bootstrap

# Apply database updates
docker run --rm -v ~/.config/promgen:/etc/promgen/ line/promgen migrate

# Create initial login user. This is the same as the default django-admin command
# https://docs.djangoproject.com/en/1.10/ref/django-admin/#django-admin-createsuperuser
docker run --rm -it -v ~/.config/promgen:/etc/promgen/ line/promgen createsuperuser
```

```shell
# Run Promgen web worker. This is typically balanced behind an NGINX instance
# 指定ALLOWED_HOSTS='*',默认为["localhost","127.0.0.1"]，否则通过ip无法登录
docker run --rm -p 8000:8000 -e DEBUG=1 -e ALLOWED_HOSTS='*' -v ~/.config/promgen:/etc/promgen/ line/promgen

# Run Promgen celery worker. Make sure to run it on the same machine as your Prometheus server to manage the config settings
docker run --rm -v ~/.config/promgen:/etc/promgen/ -v /etc/prometheus:/etc/prometheus line/promgen worker

# Or if using docker-compose you can spin up a complete test environment
docker-compose up -d
# Create initial user
docker-compose run web createsuperuser
```

### 使用

