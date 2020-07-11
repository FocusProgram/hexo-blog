---
title: Lvs+Keepalived+Nginx+FastDFS分布式文件系统高可用集群搭建
date: 2020-07-11 16:40:54
tags: 
 - Docker 
 - FastDFS
categories: Devops
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/FastDFS.png
---

高可用文件服务器架构设计图

---

![](https://gitee.com/FocusProgram/PicGo/raw/master/FastDFS.png)

所需服务器配置

---

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191127143513.png)

> ==192.168.80.128==       和 ==192.168.80.129==两台服务器上搭建Nginx+Keepalived

---

基础软件安装

```
$ yum install gcc openssl-devel libnl libnl-devel libnfnetlink-devel net-tools vim wget lrzsz lsof -y
```

> Keepalived源码包安装

下载安装Keepalived

```
$ mkdir /data && cd /data

$ wget https://www.keepalived.org/software/keepalived-2.0.19.tar.gz

$ tar -zxvf keepalived-2.0.19.tar.gz

$ cd /keepalived-2.0.19

$ ./configure

$ make && make install
```

keepalived配置

```
$ mkdir /etc/keepalived

$ cp /data/keepalived-2.0.19/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
```

开机启动项

```
$ cp /data/keepalived-2.0.19/keepalived/etc/init.d/keepalived /etc/rc.d/init.d/

$ cp /data/keepalived-2.0.19/keepalived/etc/sysconfig/keepalived /etc/sysconfig/

$ cp /usr/local/sbin/keepalived /usr/sbin/

$ chkconfig –add keepalived

$ chkconfig keepalived on
```

> Keepalived从yum源安装

```
$ yum install -y keepalived
```

> 服务命令（启动、重启、关闭）

```
$ /etc/init.d/keepalived start 启动
 
$ /etc/init.d/keepalived restart 重启

$ /etc/init.d/keepalived stop 停止
```

安装ipvsadm（用于查看lvs转发及代理情况的工具）

```
$ yum install ipvsadm -y
```

查看统计

```
#查看当前配置的虚拟服务和各个RS的权重
$ ipvsadm -Ln
#查看当前ipvs模块中记录的连接（可用于观察转发情况）
$ ipvsadm -lnc
#查看ipvs模块的转发情况统计
$ ipvsadm -Ln --stats | --rate
```

lvs超时配置
```
#查看lvs的超时时间
$ ipvsadm -L --timeout

#优化连接超时时间
$ ipvsadm --set 1 10 300
```
lvs监控真实服务测试

```
#查看最新的虚拟ip对应的RealServer的情况
$ ipvsadm -l
```

配置Keepalived

> 配置Master

```
$ cd /etc/keepalived

#备份默认的keepalived配置
$ mv keepalived.conf keepalived-back.conf

$ vim keepalived.conf
```

添加以下配置:

```
global_defs {
   # 这里配置只能发送邮件到本机
   notification_email {
         root@localhost
   }
   notification_email_from root@localhost
   smtp_server 127.0.0.1
   smtp_connection_timeout 30
   router_id LVS_DEVEL  # 设置lvs的id，在一个网络内应该是唯一的
}
vrrp_instance VI_1 {
    state MASTER   #指定Keepalived的角色，MASTER为主，BACKUP为备 记得大写
    interface ens33  #网卡id 不同的电脑网卡id会有区别 可以使用:ip a查看
    virtual_router_id 51  #虚拟路由编号，主备要一致
    priority 100  #定义优先级，数字越大，优先级越高，主DR必须大于备用DR
    advert_int 1  #检查间隔，默认为1s
    authentication {   #这里配置的密码最多为8位，主备要一致，否则无法正常通讯
        auth_type PASS
        auth_pass 159357
    }
    virtual_ipaddress {
        192.168.80.80  #定义虚拟IP(VIP)为192.168.1.200，可多设，每行一个
    }
}
# 定义对外提供服务的LVS的VIP以及port
virtual_server 192.168.80.80 80 {
    delay_loop 6 # 设置健康检查时间，单位是秒
    lb_algo rr # 设置负载调度的算法为wlc
    lb_kind DR # 设置LVS实现负载的机制，有NAT、TUN、DR三个模式
    nat_mask 255.255.255.0
    persistence_timeout 0
    protocol TCP
    real_server 192.168.80.128 80 {  # 指定real server1的IP地址
        weight 3   # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 192.168.80.129 80 {  # 指定real server2的IP地址
        weight 3  # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
     }
}

```

> 配置Backup

```
$ cd /etc/keepalived

#备份默认的keepalived配置
$ mv keepalived.conf keepalived-back.conf

$ vim keepalived.conf
```

添加以下配置:

```
global_defs {
   # 这里配置只能发送邮件到本机
   notification_email {
         root@localhost
   }
   notification_email_from root@localhost
   smtp_server 127.0.0.1
   smtp_connection_timeout 30
   router_id LVS_DEVEL  # 设置lvs的id，在一个网络内应该是唯一的
}
vrrp_instance VI_1 {
    state BACKUP   #指定Keepalived的角色，MASTER为主，BACKUP为备 记得大写
    interface ens33  #网卡id 不同的电脑网卡id会有区别 可以使用:ip a查看
    nopreempt #不与主机MASTER抢占VIP资源
    virtual_router_id 51  #虚拟路由编号，主备要一致
    priority 90 #定义优先级，数字越大，优先级越高，主DR必须大于备用DR
    advert_int 1  #检查间隔，默认为1s
    authentication {   #这里配置的密码最多为8位，主备要一致，否则无法正常通讯
        auth_type PASS
        auth_pass 159357
    }
    virtual_ipaddress {
        192.168.80.80  #定义虚拟IP(VIP)为192.168.1.200，可多设，每行一个
    }
}
# 定义对外提供服务的LVS的VIP以及port
virtual_server 192.168.80.80 80 {
    delay_loop 6 # 设置健康检查时间，单位是秒
    lb_algo rr # 设置负载调度的算法为wlc
    lb_kind DR # 设置LVS实现负载的机制，有NAT、TUN、DR三个模式
    nat_mask 255.255.255.0
    persistence_timeout 0
    protocol TCP
    real_server 192.168.80.128 80 {  # 指定real server1的IP地址
        weight 3   # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 192.168.80.129 80 {  # 指定real server2的IP地址
        weight 3  # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
     }
}

```

配置Keepalived出现问题时发送邮件


编写脚本sendmail.pl放在/etc/keepalived 中
```
$ cd /etc/keepalived

$ vim sendmail.pl

$ chmod 755 sendmail.pl
```

添加如下内容：

```
#!/usr/bin/perl -w  
use Net::SMTP_auth;  
use strict;  
my $mailhost = 'smtp.163.com';  
my $mailfrom = 'email@163.com';  
my @mailto   = ('email@163.com');  
my $subject  = 'keepalived up on backup';  
my $text = "Keepalived服务器宕机！";    
my $user   = 'email@163.com';  
my $passwd = '*******';   #注意是要填写客户端授权的密码
&SendMail();  
##############################  
# Send notice mail  
##############################  
sub SendMail() {  
    my $smtp = Net::SMTP_auth->new( $mailhost, Timeout => 120, Debug => 1 )  
      or die "Error.\n";  
    $smtp->auth( 'LOGIN', $user, $passwd );  
    foreach my $mailto (@mailto) {  
        $smtp->mail($mailfrom);  
        $smtp->to($mailto);  
        $smtp->data();  
        $smtp->datasend("To: $mailto\n");  
        $smtp->datasend("From:$mailfrom\n");  
        $smtp->datasend("Subject: $subject\n");  
        $smtp->datasend("\n");  
        $smtp->datasend("$text\n\n");   
        $smtp->dataend();  
    }  
    $smtp->quit;  
}  
```

Keepalived配置文件修改内容，注：我把脚本放到了与配置文件同级目录下，添加一段：

```
global_defs {
   # 这里配置只能发送邮件到本机
   notification_email {
         root@localhost
   }
   notification_email_from root@localhost
   smtp_server 127.0.0.1
   smtp_connection_timeout 30
   router_id LVS_DEVEL  # 设置lvs的id，在一个网络内应该是唯一的
}

vrrp_script chk_nginx {
    script "/etc/keepalived/ck_ng.sh"
    interval 2
    weight -5
    fall 3
    rise 2
}

vrrp_sync_group VG_1 {
   group {
      VI_1
}
 #节点变为master时执行
 notify_master /etc/keepalived/sendmail.pl
}

vrrp_instance VI_1 {
    state MASTER   #指定Keepalived的角色，MASTER为主，BACKUP为备 记得大写
    interface ens33  #网卡id 不同的电脑网卡id会有区别 可以使用:ip a查看
    virtual_router_id 51  #虚拟路由编号，主备要一致
    priority 100  #定义优先级，数字越大，优先级越高，主DR必须大于备用DR
    advert_int 1  #检查间隔，默认为1s
    authentication {   #这里配置的密码最多为8位，主备要一致，否则无法正常通讯
        auth_type PASS
        auth_pass 159357
    }
    virtual_ipaddress {
        192.168.80.80  #定义虚拟IP(VIP)为192.168.1.200，可多设，每行一个
    }
}
# 定义对外提供服务的LVS的VIP以及port
virtual_server 192.168.80.80 80 {
    delay_loop 6 # 设置健康检查时间，单位是秒
    lb_algo rr # 设置负载调度的算法为wlc
    lb_kind DR # 设置LVS实现负载的机制，有NAT、TUN、DR三个模式
    nat_mask 255.255.255.0
    persistence_timeout 0
    protocol TCP
    real_server 192.168.80.128 80 {  # 指定real server1的IP地址
        weight 3   # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
    }
    real_server 192.168.80.129 80 {  # 指定real server2的IP地址
        weight 3  # 配置节点权值，数字越大权重越高
        TCP_CHECK {
        connect_timeout 10
        nb_get_retry 3
        delay_before_retry 3
        connect_port 80
        }
     }
}
```

可进入该脚本目录，直接执行脚本，看看是否发送邮件成功；若失败，安装Net::SMTP_auth模块 ，安装方法：

```
$ yum -y install perl-CPAN 

$ perl -MCPAN -e shell

capn > install Net::SMTP_auth
```

> Nginx源码包安装

安装c++依赖库

```
$ yum install -y gcc-c++
```

下载依赖文件压缩包

```
$ cd /data

$ wget http://nginx.org/download/nginx-1.16.1.tar.gz

$ wget http://www.openssl.org/source/openssl-1.1.0f.tar.gz

$ wget http://zlib.NET/zlib-1.2.11.tar.gz

$ wget ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.40.tar.gz
```

解压资源

```
$ cd /data

$ tar -xf zlib-1.2.11.tar.gz

$ tar -xf openssl-1.1.0f.tar.gz

$ tar -xf pcre-8.40.tar.gz

$ tar -xf nginx-1.12.1.tar.gz
```

设置权限

```
$ chown -R root:root ./data
```

> 编译安装

安装zlib
```
$ cd /data/zlib-1.2.11/

$ ./configure

$ make && make install
```

安装openssl(用于配置ssl证书)

```
$ cd /data/openssl-1.1.0f/

$./config

$ make && make install
```

安装pcre

```
$ cd /data/pcre-8.40/

$ ./configure

$ make && make install
```

安装nginx

```
$ cd /data/nginx-1.16.1/

$ ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_stub_status_module --with-pcre=/usr/local/src/pcre-8.40 --with-zlib=/usr/local/src/zlib-1.2.11 --with-openssl=/usr/local/src/openssl-1.1.0f

$ make && make install
```

> Nginx系统自带安装包安装

查看gcc相关的安装包

```
$ yum list gcc*
```

安装依赖包

```
$ yum install -y gcc-c++ openssl openssl-devel zlib zlib-devel pcre pcre-devel
```

下载nginx

```
$ cd /data

$ wget http://nginx.org/download/nginx-1.16.1.tar.gz
```

安装nginx

```
$ cd /data

$ tar -zxvf nginx-1.16.1.tar.gz

$ cd /nginx-1.16.1

$ ./configure --prefix=/usr/local/nginx --with-http_ssl_module --with-http_stub_status_module 

$ make && make install
```

配置Nginx开机自启

```
$ vim /etc/init.d/nginx
```

添加配置如下：
```
#!/bin/bash
	#
	# chkconfig: - 85 15
	# description: Nginx is a World Wide Web server.
	# processname: nginx
	
	nginx=/usr/local/nginx/sbin/nginx
	conf=/usr/local/nginx/conf/nginx.conf
	case $1 in
	start)
	echo -n "Starting Nginx"
	$nginx
	echo " done"
	;;
	stop)
	echo -n "Stopping Nginx"
	$nginx -s stop
	echo " done"
	;;
	test)
	$nginx -t -c $conf
	;;
	reload)
	echo -n "Reloading Nginx"
	$nginx -s reload
	echo " done"
	;;
	restart)
	sh $0 stop
	sh $0 start
	;;
	show)
	ps -aux|grep nginx
	;;
	*)
	echo -n "Usage: $0 {start|restart|reload|stop|test|show}"
	;;
	esac
```

配置文件nginx的权限

```
$ chmod 755 /etc/init.d/nginx
```

设置开机自启动

```
$ chkconfig nginx on
```


> 配置防火墙方式一（弃用firewalld使用iptables）

停止和禁用firewalld

```
$ systemctl stop firewalld && systemctl disable firewalld
```

安装iptables相关组件

```
$ yum install -y iptables-services iptables-devel.x86_64 iptables.x86_64

$ systemctl enable iptables #启用iptables

$ systemctl start iptables  #启动iptables

$ systemctl status iptables #查看iptables状态
```

> ==192.168.80.128== 和 ==192.168.80.129==配置防火墙

keepalived服务器下的配置

```
$ vim /etc/sysconfig/iptables

#允许vrrp多播心跳(如果防火墙开启，这里不配置这个，就会出现裂脑)
-I INPUT -p vrrp -j ACCEPT
#开启80端口的访问(如果防火墙开启，不配置这个，vip的80端口将无法正常访问)
-I INPUT -p tcp --dport 80 -j ACCEPT
```

nginx服务器下配置

```
#nginx默认监听的80端口 这里直接开启80端口的外网访问(不开启外网将无法正常反问对应服务器的nginx)
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
```

重启防火墙

```
systemctl restart iptables.service
```

> 配置防火墙方式一（禁用防火墙，生产环境不推荐）

```
$ systemctl stop iptables.service

$ systemctl stop firewalld
```

配置Nginx

```
$ cd /etc/init.d/

$ vim realserver
```

添加如下配置

```
#虚拟的vip 根据自己的实际情况定义
SNS_VIP=192.168.80.80
/etc/rc.d/init.d/functions
case "$1" in
start)
       ifconfig lo:0 $SNS_VIP netmask 255.255.255.255 broadcast $SNS_VIP
       /sbin/route add -host $SNS_VIP dev lo:0
       echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
       sysctl -p >/dev/null 2>&1
       echo "RealServer Start OK"
       ;;
stop)
       ifconfig lo:0 down
       route del $SNS_VIP >/dev/null 2>&1
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
       echo "RealServer Stoped"
       ;;
*)
       echo "Usage: $0 {start|stop}"
       exit 1
esac
exit 0
```

保存并设置脚本的执行权限

```
$ chmod 755 /etc/init.d/realserver
```

因为realserver脚本中用到了/etc/rc.d/init.d/functions，所以一并设置权限

```
$ chmod 755 /etc/rc.d/init.d/functions
```

执行脚本

```
$ service realserver start
```

设置开机自启

```
$ vim /etc/rc.d/rc.local

#添加如下内容
exec service realserver start

#查看文件操作权限

$ ll /etc/rc.d/rc.local

#设置文件操作权限

$ chmod +x /etc/rc.d/rc.local

或

$ chmod 755 /etc/rc.d/rc.local
```

查看是否配置成功

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191127234906.png)

> 启动Keepalived

```
$ /etc/init.d/keepalived start #启动

$ /etc/init.d/keepalived restart #重启

$ /etc/init.d/keepalived stop # 停止
```

检查主keepalived 启动后的配置情况
ip a
如果网卡下出现192.168.80.80（VIP）说明主已经启动成功

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191127235938.png)


检查备keepalived 启动后的配置情况
ip a
备服务器的网卡下没有出现192.168.80.80（VIP）的ip，说明备服务正常
注:如果这里也出现了VIP，那么说明裂脑了，需要检查防火墙是否配置正确；是否允许了vrrp的多播通讯

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191128000041.png)


> FastDFS集群搭建

==192.168.80.130== 、==192.168.80.131== 、==192.168.80.132== 、==192.168.80.133== 、
==192.168.80.134== 、 ==192.168.80.135== 上安装FastDFS所需的安装环境

编译环境

```
yum install git gcc gcc-c++ make automake autoconf libtool pcre pcre-devel zlib zlib-devel openssl-devel wget vim git -y
```

配置防火墙开放相应的端口(禁用firewalld采用iptables配置)

```
$ systemctl stop firewalld && systemctl disable firewalld

$ yum install -y iptables-services iptables-devel.x86_64 iptables.x86_64

$ vim /etc/sysconfig/iptables
```

添加如何配置，开放相应的端口

```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT

-A INPUT -p tcp -m state --state NEW -m tcp --dport 22122 -j ACCEPT

-A INPUT -p tcp -m state --state NEW -m tcp --dport 23000 -j ACCEPT
```

启用iptables并设置开机自启

```
$ systemctl start iptables && systemctl enable iptables
```

查看防火墙对外开放的端口

```
$ iptables -L -n
```

创建下载文件目录和存储文件目录

```
$ mkdir -p /data/fastdfs/{data,store_path}

$ mkdir -p /data/soft && cd /data/soft
```

安装libfatscommon

```
$ git clone https://github.com/happyfish100/libfastcommon.git --depth 1

#$ cd libfastcommon/

$ 
./make.sh && ./make.sh install #编译安装
```

安装FastDFS

```
$ cd ../ #返回上一级目录

$ git clone https://github.com/happyfish100/fastdfs.git --depth 1

$ cd fastdfs/

$ ./make.sh && ./make.sh install #编译安装
```

配置文件准备

```
#编写所需文件执行脚本

$ touch config.sh && chmod 777 config.sh && vim config.sh
```

#添加如下内容

```
#! /bin/bash

#Tracker文件
cp /etc/fdfs/tracker.conf.sample /etc/fdfs/tracker.conf

#Storage文件
cp /etc/fdfs/storage.conf.sample /etc/fdfs/storage.conf

#客户端文件(测试)
cp /etc/fdfs/client.conf.sample /etc/fdfs/client.conf

#供nginx访问使用
cp /data/soft/fastdfs/conf/http.conf /etc/fdfs/

#供nginx访问使用
cp /data/soft/fastdfs/conf/mime.types /etc/fdfs/
```

安装fastdfs-nginx-module

```
$ cd ../ #返回上一级目录

$ git clone https://github.com/happyfish100/fastdfs-nginx-module.git --depth 1

$ cp /data/soft/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs
```

安装nginx

```
#下载nginx压缩包
$ wget http://nginx.org/download/nginx-1.16.1.tar.gz

#解压
$ tar -zxvf nginx-1.16.1.tar.gz 

$ cd nginx-1.16.1/

#添加fastdfs-nginx-module模块
$ ./configure --add-module=/data/soft/fastdfs-nginx-module/src/ 

#编译安装
$ make && make install
```

Tracker配置

#服务器ip为 ==192.168.80.130== 、==192.168.80.131==
```
$ vim /etc/fdfs/tracker.conf

#需要修改的内容如下
port=22122  # tracker服务器端口（默认22122,一般不修改）
base_path=/data/fastdfs/data  # 存储日志和数据的根目录
```

Storage配置

#服务器ip为 ==192.168.80.132== 、==192.168.80.133== 、==192.168.80.134== 、==192.168.80.135==

```
$ vim /etc/fdfs/storage.conf

#需要修改的内容如下
port=23000  # storage服务端口（默认23000,一般不修改）

# 数据和日志文件存储根目录
base_path=/data/fastdfs/data

 # 第一个存储目录
store_path0=/data/fastdfs/store_path
tracker_server=192.168.80.130:22122  # 服务器1
tracker_server=192.168.80.131:22122  # 服务器2

# http访问文件的端口(默认8888,看情况修改,和nginx中保持一致)
http.server_port=80  
```

client测试

#服务器ip为 ==192.168.80.132== 、==192.168.80.133== 、==192.168.80.134== 、==192.168.80.135==

```
$ vim /etc/fdfs/client.conf

#需要修改的内容如下
base_path=/home/moe/dfs
tracker_server=192.168.80.130:22122  # 服务器1
tracker_server=192.168.80.131:22122  # 服务器2

#保存后测试,返回ID表示成功 如：group1/M00/00/00/xx.tar.gz
fdfs_upload_file /etc/fdfs/client.conf /usr/local/src/nginx-1.15.4.tar.gz
```


配置nginx访问

#服务器ip为 ==192.168.80.132== 、==192.168.80.133== 、==192.168.80.134== 、==192.168.80.135==
```
$ vim /etc/fdfs/mod_fastdfs.conf

#需要修改的内容如下
tracker_server=192.168.80.131:22122  # 服务器1
tracker_server=192.168.80.132:22122  # 服务器2

url_have_group_name=true
store_path0=/data/fastdfs/data

#配置nginx.config
$ vim /usr/local/nginx/conf/nginx.conf

#添加如下配置
server {
    listen       80;    ## 该端口为storage.conf中的http.server_port相同
    server_name  localhost;
    location ~/group[0-9]/ {
        ngx_fastdfs_module;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
    root   html;
    }
}
```

> 配置开机自启

Nginx开机自启

```
$ vim /etc/init.d/nginx
```

添加配置如下：
```
#!/bin/bash
#
# chkconfig: - 85 15
# description: Nginx is a World Wide Web server.
# processname: nginx

nginx=/usr/local/nginx/sbin/nginx
conf=/usr/local/nginx/conf/nginx.conf
case $1 in
start)
echo -n "Starting Nginx"
$nginx
echo " done"
;;
stop)
echo -n "Stopping Nginx"
$nginx -s stop
echo " done"
;;
test)
$nginx -t -c $conf
;;
reload)
echo -n "Reloading Nginx"
$nginx -s reload
echo " done"
;;
restart)
sh $0 stop
sh $0 start
;;
show)
ps -aux|grep nginx
;;
*)
echo -n "Usage: $0 {start|restart|reload|stop|test|show}"
;;
esac
```

配置文件nginx的权限

```
$ chmod 755 /etc/init.d/nginx
```

设置开机自启动

```
$ chkconfig nginx on
```

FastDFS配置开机自启

```
$ chmod +x /etc/rc.d/rc.local 

$ vim /etc/rc.d/rc.local
```
添加如下命令

```
exec /usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf start

exec /usr/bin/fdfs_storaged /etc/fdfs/storage.conf start
```

操作FastDFS常用命令

```
$ /etc/init.d/fdfs_trackerd start #启动tracker服务
$ /etc/init.d/fdfs_trackerd restart #重启动tracker服务
$ /etc/init.d/fdfs_trackerd stop #停止tracker服务
$ chkconfig fdfs_trackerd on #自启动tracker服务

$ /etc/init.d/fdfs_storaged start #启动storage服务
$ /etc/init.d/fdfs_storaged restart #重动storage服务
$ /etc/init.d/fdfs_storaged stop #停止动storage服务
$ chkconfig fdfs_storaged on #自启动storage服务

$ /usr/local/nginx/sbin/nginx #启动nginx
$ /usr/local/nginx/sbin/nginx -s reload #重启nginx
$ /usr/local/nginx/sbin/nginx -s stop #停止nginx
```

检测集群

```
$ /usr/bin/fdfs_monitor /etc/fdfs/storage.conf
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/Tracker-one.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/Tracker-two.png)

客户端测试上传

```
#保存后测试,返回ID表示成功 如：group1/M00/00/00/xx.tar.gz

$ fdfs_upload_file /etc/fdfs/client.conf /data/soft/nginx-1.16.1.tar.gz

$ /usr/bin/fdfs_test /etc/fdfs/client.conf upload /data/soft/nginx-1.16.1.tar.gz
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191130184341.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191130184409.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191130184433.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191130184703.png)

若四台机器均可正常访问图片,至此两台Tracker+四台Storage正式搭建成功

> 配置文件访问的负载均衡和高可用

在==192.168.80.128== 和 ==192.168.80.129==上配置nginx配置文件

```
$ vim /usr/local/nginx/conf/nginx.conf
```

修改配置如下：

```
#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #设置 group1 的服务器
    upstream fdfs_group1 {
          server 192.168.80.132:80 weight=1 max_fails=2 fail_timeout=30s;
          server 192.168.80.133:80 weight=1 max_fails=2 fail_timeout=30s;
          server 192.168.80.134:80 weight=1 max_fails=2 fail_timeout=30s;
          server 192.168.80.135:80 weight=1 max_fails=2 fail_timeout=30s;
    }
    

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        #location / {
        #    root   html;
        #    index  index.html index.htm;
        #}

    	#设置 group 的负载均衡参数
    	location /group1/M00 {
    	      proxy_set_header Host $host;
    	      proxy_set_header X-Real-IP $remote_addr;
    	      proxy_pass http://fdfs_group1;
    	}

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```

通过VIP主机访问，访问成功，至此LVS+Keepalived+Nginx+FastDFS高可用集群全部搭建成功

![](https://gitee.com/FocusProgram/PicGo/raw/master/20191130191442.png)

> Fast分组集群

修改==192.168.80.132== 和 ==192.168.80.133== 的Storage.conf如下：

```
base_path=/data/fastdfs/data
store_path0=/data/fastdfs/store_path
tracker_server=192.168.80.130:22122
tracker_server=192.168.80.131:22122
group_name=group1
```

修改==192.168.80.132== 和 ==192.168.80.133== 的mod_fastdfs.conf如下：

```
base_path=/data/fastdfs/storage
#保留默认值也可以
connect_timeout=10                       
tracker_server=192.168.80.130:22122
tracker_server=192.168.80.131:22122
url_have_group_name = true                #url中是否加上group名
store_path0=/data/fastdfs/store_path
group_name=group1                        #当前storage所属的组名
group_count = 2                    #组的数量，示例中共两组：group1、group2

[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/data/fastdfs/store_path

[group2]
group_name=group2
storage_server_port=23000
store_path_count=1
store_path0=/data/fastdfs/store_path
```

修改==192.168.80.134== 和 ==192.168.80.135== 的Storage.conf如下：

```
base_path=/data/fastdfs/data
store_path0=/data/fastdfs/store_path
tracker_server=192.168.80.130:22122
tracker_server=192.168.80.131:22122
group_name=group2
```

修改==192.168.80.134== 和 ==192.168.80.135== 的mod_fastdfs.conf如下：

```
base_path=/data/fastdfs/storage
#保留默认值也可以
connect_timeout=10                       
tracker_server=192.168.80.130:22122
tracker_server=192.168.80.131:22122
url_have_group_name = true                #url中是否加上group名
store_path0=/data/fastdfs/store_path
group_name=group2                        #当前storage所属的组名
group_count = 2                    #组的数量，示例中共两组：group1、group2

[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/data/fastdfs/store_path

[group2]
group_name=group2
storage_server_port=23000
store_path_count=1
store_path0=/data/fastdfs/store_path
```

修改==192.168.80.128== 和 ==192.168.80.129== 的nginx.conf如下：

```

#user  nobody;
worker_processes  1;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    #设置 group1 的服务器
    upstream fdfs_group1 {
          server 192.168.80.132:80 weight=1 max_fails=2 fail_timeout=30s;
          server 192.168.80.133:80 weight=1 max_fails=2 fail_timeout=30s;
    }

    #设置 group1 的服务器
    upstream fdfs_group2 {
	  server 192.168.80.134:80 weight=1 max_fails=2 fail_timeout=30s;
	  server 192.168.80.135:80 weight=1 max_fails=2 fail_timeout=30s;
    }
    

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';

    #access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    server {
        listen       80;
        server_name  192.168.80.128;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        #location / {
        #    root   html;
        #    index  index.html index.htm;
        #}

	#设置 group1 的负载均衡参数
	location /group1/M00 {
	      proxy_set_header Host $host;
	      proxy_set_header X-Real-IP $remote_addr;
	      proxy_pass http://fdfs_group1;
	}

	#设置 group2 的负载均衡参数
	location /group2/M00 {
	      proxy_set_header Host $host;
	      proxy_set_header X-Real-IP $remote_addr;
	      proxy_pass http://fdfs_group2;
	}
	                                                          

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }


    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}


    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}

```

> Nginx追加https模块

查看nginx原有的模块

```
$ /usr/local/nginx/sbin/nginx -V
```

添加Https模块

```
$ cd /data/nginx-1.16.1

$ ./configure --with-http_stub_status_module --with-http_ssl_module
```

编译不覆盖安装

```
$ make

$ cp ./objs/nginx /usr/local/nginx/sbin/
```