---
title: Docker+Nginx+FastDFS分布式文件系统搭建
date: 2020-07-11 16:40:39
tags: 
 - Docker 
 - FastDFS
categories: Devops
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/20191107104303.png
---

FastDFS介绍

---
1.1   什么是FastDFS

         FastDFS是用c语言编写的一款开源的分布式文件系统。FastDFS为互联网量身定制，充分考虑了冗余备份、负载均衡、线性扩容等机制，并注重高可用、高性能等指标，使用FastDFS很容易搭建一套高性能的文件服务器集群提供文件上传、下载等服务。

1.2   FastDFS架构

         FastDFS架构包括 Tracker server和Storage server。客户端请求Tracker server进行文件上传、下载，通过Trackerserver调度最终由Storage server完成文件上传和下载。

         Trackerserver作用是负载均衡和调度，通过Trackerserver在文件上传时可以根据一些策略找到Storageserver提供文件上传服务。可以将tracker称为追踪服务器或调度服务器。

         Storageserver作用是文件存储，客户端上传的文件最终存储在Storage服务器上，Storage server没有实现自己的文件系统而是利用操作系统 的文件系统来管理文件。可以将storage称为存储服务器。

	Tracker 集群
	
	FastDFS集群中的Trackerserver可以有多台，Trackerserver之间是相互平等关系同时提供服务，Trackerserver不存在单点故障。客户端请求Trackerserver采用轮询方式，如果请求的tracker无法提供服务则换另一个tracker。
	
	Storage 集群
	
	Storage集群采用了分组存储方式。storage集群由一个或多个组构成，集群存储总容量为集群中所有组的存储容量之和。一个组由一台或多台存储服务器组成，组内的Storage server之间是平等关系，不同组的Storageserver之间不会相互通信，同组内的Storageserver之间会相互连接进行文件同步，从而保证同组内每个storage上的文件完全一致的。一个组的存储容量为该组内存储服务器容量最小的那个，由此可见组内存储服务器的软硬件配置最好是一致的。
	
	采用分组存储方式的好处是灵活、可控性较强。比如上传文件时，可以由客户端直接指定上传到的组也可以由tracker进行调度选择。一个分组的存储服务器访问压力较大时，可以在该组增加存储服务器来扩充服务能力（纵向扩容）。当系统容量不足时，可以增加组来扩充存储容量（横向扩容）。

搭建FastDFS文件系统

---

准备工作-关闭防火墙

```
$ systemctl stop firewalld & systemctl disable firewalld

$ vim /etc/sysconfig/selinux
  
  SELINUX=disabled
```

准备工作-编译环境（FastDFS是C语言开发，安装FastDFS需要先将官网下载的源码进行编译，编译依赖gcc环境，如果没有gcc环境，需要安装gcc）

```
$ yum install git gcc gcc-c++ make automake autoconf libtool pcre pcre-devel zlib zlib-devel openssl-devel wget vim -y
```

准备工作-Git

```
$ yum install git
```

1.下载镜像

```
$ docker search fastdfs

$ docker pull season/fastdfs
```

2.创建文件存储目录

```
$ mkdir -p /data/fdfs
```

3.安装libfatscommon

```
$ git clone https://github.com/happyfish100/libfastcommon.git --depth 1

$ cd libfastcommon/

$ ./make.sh && ./make.sh install #编译安装
```

4.安装FastDFS

```
$ git clone https://github.com/happyfish100/fastdfs.git --depth 1

$ cd fastdfs/

$ ./make.sh && ./make.sh install #编译安装
```

5.安装fastdfs-nginx-module

```
$ git clone https://github.com/happyfish100/fastdfs-nginx-module.git --depth 1
```

6.安装Nginx

```
$ wget http://nginx.org/download/nginx-1.15.4.tar.gz #下载nginx压缩包

$ tar -zxvf nginx-1.15.4.tar.gz #解压

$ cd nginx-1.15.4/

#添加fastdfs-nginx-module模块
$ ./configure --add-module=/data/fdfs/fastdfs-nginx-module/src/ 

$ make && make install #编译安装

$ cd /usr/local 查看nginx是否安装成功，成功则显示nginx目录
```

7.创建FastDFS存储目录

```
$ mkdir -p /data/fdfs/{tracker_data,storage_data,store_path,fdfs_conf}
```

8.启动一个临时的tracker 拷贝storage.conf tracker.conf 至/var/fdfs/fdfs_conf

```
docker run  -d --name tracker  --net=host season/fastdfs tracker
docker ps 

//查询到容器id
docker cp 容器ID:/fdfs_conf/tracker.conf /data/fdfs/fdfs_conf/
docker cp 容器ID:/fdfs_conf/storage.conf /data/fdfs/fdfs_conf/
```

9.单机配置

```
tracker配置

$ vim /etc/fdfs/tracker.conf
#需要修改的内容如下
port=22122                  # tracker服务器端口（默认22122,一般不修改）
base_path=/data/fdfs/storage  # 存储日志和数据的根目录
```

```
storage配置

$ vim /etc/fdfs/storage.conf
#需要修改的内容如下
port=23000  # storage服务端口（默认23000,一般不修改）
base_path=/data/fdfs/storage  # 数据和日志文件存储根目录
store_path0=/data/fdfs/store_path # 第一个存储目录
tracker_server=192.168.199.120:22122  # tracker服务器IP和端口
http.server_port=80  # http访问文件的端口(默认8888,看情况修改,和nginx中保持一致)
```

```
client测试

vim /etc/fdfs/client.conf
#需要修改的内容如下
base_path=/fastdfs/storage
tracker_server=192.168.199.120:22122    #tracker服务器IP和端口
#保存后测试,返回ID表示成功 如：group1/M00/00/00/xx.tar.gz
fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/nginx-1.15.4.tar.gz
```

```
配置nginx访问

vim /etc/fdfs/mod_fastdfs.conf
#需要修改的内容如下
tracker_server=192.168.199.120:22122  #tracker服务器IP和端口
url_have_group_name=true #是否使用group_name为前缀
store_path0=/data/fdfs/store_path

#配置nginx.config
vim /usr/local/nginx/conf/nginx.conf
#添加如下配置
user root;              ##配置访问权限

server {
    listen       80;    ## 该端口为storage.conf中的http.server_port相同
    server_name  192.168.199.120;
    # location ~/group[0-9]/ {
      location ~/kongqi/{
        root /data/fdfs/storage_path/data;
        ngx_fastdfs_module;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    root   html;
    }
}
```

10.分布式部署

```
tracker配置

vim /etc/fdfs/tracker.conf
#需要修改的内容如下
port=22122                  # tracker服务器端口（默认22122,一般不修改）
base_path=/data/fdfs/storage  # 存储日志和数据的根目录
```

```
storage配置

vim /etc/fdfs/storage.conf
#需要修改的内容如下
port=23000  # storage服务端口（默认23000,一般不修改）
base_path=/data/fdfs/storage  # 数据和日志文件存储根目录
store_path0=/data/fdfs/store_path # 第一个存储目录
tracker_server=192.168.199.120:22122  # 服务器1
tracker_server=192.168.199.121:22122  # 服务器2
tracker_server=192.168.199.122:22122  # 服务器3
http.server_port=80  # http访问文件的端口(默认8888,看情况修改,和nginx中保持一致)
```

```
client测试

vim /etc/fdfs/client.conf
#需要修改的内容如下
base_path=/data/fdfs/store_path  # 数据和日志文件存储根目录
tracker_server=192.168.199.120:22122  # 服务器1
tracker_server=192.168.199.121:22122  # 服务器2
tracker_server=192.168.199.122:22122  # 服务器3
#保存后测试,返回ID表示成功 如：group1/M00/00/00/xx.tar.gz
fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/nginx-1.15.4.tar.gz
```

```
配置nginx访问

vim /etc/fdfs/mod_fastdfs.conf
#需要修改的内容如下
tracker_server=192.168.199.120:22122  # 服务器1
tracker_server=192.168.199.121:22122  # 服务器2
tracker_server=192.168.199.122:22122  # 服务器3
url_have_group_name=true
store_path0=/data/fdfs/store_path # 第一个存储目录

#配置nginx.config
vim /usr/local/nginx/conf/nginx.conf
#添加如下配置
user root;              ##配置访问权限

server {
    listen       80;    ## 该端口为storage.conf中的http.server_port相同
    server_name  192.168.199.120;
    # location ~/group[0-9]/ {
      location ~/kongqi/{
        root /data/fdfs/storage_path/data;
        ngx_fastdfs_module;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    root   html;
    }
}
```

11.编写配置文件脚本(/etc/fdfs目录中的所有文件是nginx 整合fastdfs-nginx-module所用到的配置文件)

```
#! /bin/bash

cp /data/fdfs/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs

cp -r /data/fdfs/fastdfs/conf/* /etc/fdfs/

cp -r /data/fdfs/fdfs_conf/* /etc/fdfs/
```

12.docker部署tracker和storage容器

```
运行tracker镜像

$ docker run -d --name trakcer \
  --restart=always \
  -v /data/fdfs/fdfs_conf/tracker.conf:/fdfs_conf/tracker.conf \
  -v /data/fdfs/tracker_data:/fastdfs/tracker/data \
  --net=host season/fastdfs tracker
```

```
运行storage镜像

$ docker run -d --name storage \
  --restart=always \
  -v /data/fdfs/fdfs_conf/storage.conf:/fdfs_conf/storage.conf \
  -v /data/fdfs/storage_data:/fastdfs/storage/data \
  -v /data/fdfs/store_path:/fastdfs/store_path \
  --net=host season/fastdfs storage
```

```
Tracker服务器的端口默认是22122，你可以查看是否启用端口

$ netstat -aon | grep 22122
```

```
查看tracker容器和storage容器的关联

$ docker exec -it storage /bin/bash

$ cd fdfs_conf

$ fdfs_monitor storage.conf
```

13.运行nginx进行端口监听

```
cd /usr/local/nginx/sbin

./nginx #启动Nginx

./nginx -s reload #重新加载 

./nginx -s stop #停止
```

```
设置nginx开机自启动

新增shell脚本 vi /etc/rc.d/init.d/nginx

#!/bin/sh
#
#nginx - this script starts and stops the nginx daemin
#
# chkconfig: - 85 15
# description: Nginx is an HTTP(S) server, HTTP(S) reverse \
# proxy and IMAP/POP3 proxy server
# processname: nginx
# config: /usr/local/nginx/conf/nginx.conf
# pidfile: /usr/local/nginx/logs/nginx.pid
# Source function library.
. /etc/rc.d/init.d/functions
# Source networking configuration.
. /etc/sysconfig/network
# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0
nginx="/usr/local/nginx/sbin/nginx"
prog=$(basename $nginx)
NGINX_CONF_FILE="/usr/local/nginx/conf/nginx.conf"
lockfile=/var/lock/subsys/nginx
start() {
[ -x $nginx ] || exit 5
[ -f $NGINX_CONF_FILE ] || exit 6
echo -n $"Starting $prog: "
daemon $nginx -c $NGINX_CONF_FILE
retval=$?
echo
[ $retval -eq 0 ] && touch $lockfile
return $retval
}
stop() {
echo -n $"Stopping $prog: "
killproc $prog -QUIT
retval=$?
echo
[ $retval -eq 0 ] && rm -f $lockfile
return $retval
}
restart() {
configtest || return $?
stop
start
}
reload() {
configtest || return $?
echo -n $"Reloading $prog: "
killproc $nginx -HUP
RETVAL=$?
echo
}
force_reload() {
restart
}
configtest() {
$nginx -t -c $NGINX_CONF_FILE
}
rh_status() {
status $prog
}
rh_status_q() {
rh_status >/dev/null 2>&1
}
case "$1" in
start)
rh_status_q && exit 0
$1
;;
stop)
rh_status_q || exit 0
$1
;;
restart|configtest)
$1
;;
reload)
rh_status_q || exit 7
$1
;;
force-reload)
force_reload
;;
status)
rh_status
;;
condrestart|try-restart)
rh_status_q || exit 0
;;
*)
echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
exit 2
esac

加入系统服务并开机自启动

chmod +x /etc/rc.d/init.d/nginx （设置可执行权限）

chkconfig --add nginx （添加系统服务）

chkconfig --level 35 nginx on （开机自启动）
```

14.测试是否成功部署

```
curl http://192.168.199.120/kongqi/M00/00/00/wKjHeF27C5KAZdUSAAXmEY7V_qc038.jpg
```

15.多级存储目录配置

```
同时修改此三处的配置

$ vim /data/fdfs/fdfs_conf/storage.conf

$ vim vim /etc/fdfs/storage.conf

$ vim /etc/fdfs/mod_fastdfs.conf

  store_path_count=2                 #store_path存储路径个数
  
  store_path0=/data/fdfs/store_path  #第一个存储目录
  
  store_path1=/data/fdfs/store_path1 #第二个存储目录
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191101153906.png)

```
Nginx配置如下：

        location ~/M00 {
           root /data/fdfs/store_path/data;
           ngx_fastdfs_module;
        }

        location ~/M01 {
           root /data/fdfs/store_path1/data;
           ngx_fastdfs_module;
        }

```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191101154417.png)

16.批量迁移数据出现问题解决问题

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191106094507.png)

```
$ vim /data/fdfs/fdfs_conf/storage.conf

  # 修改最大连接数
  # max concurrent connections the server supported
  # default value is 256
  # more max_connections means more memory will be used
  max_connections=10000
```

17.配置防盗链

```
$ vim /etc/fdfs/http.conf

  http.anti_steal.check_token=true #是否开启防盗链
  
  http.anti_steal.token_ttl=86400 #防盗链失效时间（单位/秒）
  
  http.anti_steal.secret_key=FastDFS1234567890 #防盗链秘钥
  
  http.anti_steal.token_check_fail=/etc/fdfs/404.jpg #防盗链检查失败时重定向图片
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191107104236.png)

```
java客户端生成token

    //unix时间戳 以秒为单位
    int ts = (int) (System.currentTimeMillis() / 1000);
    String secret_key = "FastDFS1234567890";
    String token = new String();
    token = ProtoCommon.getToken(url, ts, secret_key);
    StringBuilder sb = new StringBuilder();
    sb.append("http://192.168.199.120/");
    sb.append(url);
    sb.append("?token=").append(token);
    sb.append("&ts=").append(ts);
```
![](https://gitee.com/FocusProgram/PicGo/raw/master/20191107104303.png)
