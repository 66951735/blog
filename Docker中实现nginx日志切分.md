## Docker中实现nginx日志切分

#### 背景

- 默认情况下，`nginx`的访问日志并不会按照日期自动滚动，整个服务生命周期内，只会写一个日志文件，如果长时间运行，必然会造成日志文件非常巨大，给我们的日志清理造成了很多不便。
- 在传统的应用部署方案中，一般使用`logrotate`来实现`nginx`访问日志的切分。
- 事实上这一方案，在`Docker`容器环境下依然使用，本文就介绍下如何在`docker`环境中通过`logroate`实现`nginx`访问日志的切分。

#### 关于logrotate

- `logrotate`是一个日志回滚的工具，它会根据一些配置规则，对**日志文件进行切分并删除日志**
- `logrotate`本身并不是一个`daemon`进程，它是一个`crond`任务，按照`cron`表达式的规则定时执行
- `logrotate`在做完文件切分操作之后需要通知写日志的进程在新的日志文件继续写入，一般是发送一个信号，这个操作需要在`logrotate`的配置文件中显示指定

#### Dockerfile

```
FROM nginx

RUN mkdir log
# 安装logrotate
RUN apt-get update && apt-get -y install logrotate

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo '$TZ' > /etc/timezone

# 拷贝nginx配置文件到容器内相应目录
COPY nginx.conf /etc/nginx/nginx.conf

# 拷贝logrotate nginx配置文件
COPY kideng-game/logrotate-nginx /etc/logrotate.d/

# 拷贝crontab 定时任务
COPY kideng-game/crontab /etc/

# 开启cron定时任务并以非daemon的方式启动nginx
CMD service cron start && nginx -g 'daemon off;'
```

**注意：**这里必须以非deamon的方式启动nginx，否则容器执行完启动命令后会自动退出，关于原因参见[nginx的docker仓库](https://hub.docker.com/_/nginx/)

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gfbuowicf4j31v40acjv5.jpg)

#### nginx配置文件

```
user  nginx;
worker_processes  1;

# error日志
error_log  /log/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 设置日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $request_length $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for" $request_time '
                      '$host $ssl_protocol $ssl_cipher $remote_port '
                      '"$upstream_cache_status" "$upstream_response_length" '
                      '"$upstream_addr" "$upstream_status" "$upstream_response_time" ';
    # access日志
    access_log  /log/access.log  main;

    proxy_http_version 1.1;
    proxy_set_header Connection Keep-Alive;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;

    sendfile        on;
    keepalive_timeout  65;
    client_max_body_size 200m;
}
```

#### logrotate配置文件

```
/log/*.log {
  su nginx root # 切换用户，解决可能存在的文件权限问题，根据实际环境配置
  daily # 按天切割日志
  missingok # 忽略错误
  rotate 7 # 保留最近的7个文件
  ifempty # 空文件也切割
  create 640 nginx root # 创建640的日志文件
  sharedscripts # 多个日志文件共享配置脚本
  dateext # 按日期命名切分后的日志文件，默认为`文件名-%Y%m%d`，目前支持到小时，不支持分和秒的命名
  postrotate # 日志切分后执行的脚本，用于通知写日志的服务重新定位日志文件，nginx kill -USR1信号为重新打开日志文件
    if [ -f /var/run/nginx.pid ]; then
      kill -USR1 `cat /var/run/nginx.pid`
    fi
  endscript
}
```

#### cron配置文件

```
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
# 主要的修改在这里，默认按天执行的定时任务具体时间为早上6点25分，改成每天的23点55分执行，可能延迟
55 23    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```

#### github项目地址

[Docker中实现nginx日志切分](https://github.com/66951735/docker-nginx-logrotate)