---
title: Jenkins部署项目
date: 2020-07-11 16:44:09
tags: 
 - Docker 
 - Jenkins
categories: Devops
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/20190918161333.png
---

**Jenkins部署项目**

---
> 新建任务

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918161333.png)

> 配置任务

1.General配置

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918161450.png)

2.源码管理

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918161708.png)

3.构建环境

```
1.Delete workspace before build starts：构建前清空工作空间

2.Use secret text(s) or file(s)：使用加密文件或文本

3.Abort the build if it’s stuck：构建出现问题，则终止构建

4.Add timestamps to the Console Output：给控制台输出增加时间戳

5.Inspect build log for published Gradle build scans：检查已发布的Gradle构建扫描的构建日志

6.With Ant：用 Ant
```

4.构建触发器

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918171407.png)

==<1>.触发远程构建(例如脚本)==

```
该触发方法是指通过调用jenkins接口触发构建任务(token在任务中配置)

curl http://192.168.80.80:8080/job/springboot-quartz/build?token=123456
```

==<2>.其他工程构建后触发==

```
该触发方法可以选择在jenkins构建完某个项目时触发构建
```

==<3>.GitHub hook trigger for GITScm polling==
```
该方法是通过向github提交代码时触发
```
系统管理 》 系统设置

github中生成token

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918163851.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918163359.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918164046.png)

jenkins系统设置中配置 Hook URL

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918164248.png)

```
http://p2g1055383.iask.in --> http://192.168.80.80:8080 外网映射
```

github中配置Webhooks地址

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918164425.png)

==<4>.定时构建==

```
该构建是指在某个时刻定时去构建项目类似定时任务

不管SVN或Git中数据有无变化，均执行定时化的构建任务
```

==<5>.轮训SCM==

```
只要SVN或Git中数据有更新，则执行构建任务
```

```
日程表表达式可以参考https://blog.csdn.net/nklinsirui/article/details/95338535
```

5.构建

==<1>.Invoke Ant==

==<2>.Invoke Gradle script==

==<3>.Run with timeout==

==<4>.Set Build status to "pending"  on GitHub commit==

==<5>.执行Windows批处理命令==

==<6>.执行Shell==

==<7>.调用顶层Maven目标==

![](https://gitee.com/FocusProgram/PicGo/raw/master/20190918165242.png)

> shell脚本构建jar直接运行java -jar项目

```
#!/bin/bash

#服务名称
SERVER_NAME=quartz

# 源jar路径,mvn打包完成之后，target目录下的jar包名称，也可选择成为war包，war包可移动到Tomcat的webapps目录下运行，这里使用jar包，用java -jar 命令执行  
JAR_NAME=quartz-0.0.1-SNAPSHOT

# 源jar路径  
#/usr/local/jenkins_home/workspace--->jenkins 工作目录

#demo 项目目录
#target 打包生成jar包的目录
JAR_PATH=/data/jenkins_home/workspace/quartz/target/

# 打包完成之后，把jar包移动到运行jar包的目录
JAR_WORK_PATH=/data/run

echo "查询进程id-->$SERVER_NAME"
PID=`ps -ef | grep "$SERVER_NAME" | awk '{print $2}'`
echo "得到进程ID：$PID"
echo "结束进程"
for id in $PID
do
	kill -9 $id  
	echo "killed $id"  
done
echo "结束进程完成"

#复制jar包到执行目录
echo "复制jar包到执行目录:cp $JAR_PATH/$JAR_NAME.jar $JAR_WORK_PATH"
cp $JAR_PATH/$JAR_NAME.jar $JAR_WORK_PATH
echo "复制jar包完成"
cd $JAR_WORK_PATH
#修改文件权限
chmod 755 $JAR_NAME.jar

#后台运行jar包
BUILD_ID=dontKillMe nohup java -jar  $JAR_NAME.jar  &
```

> shell脚本构建war部署tomcat项目

```
#!/bin/bash

#防止tomcat和jenkins位于同一服务器上时tomcat启动失败
export BUILD_ID=tomcat_build_id

cd /data/jenkins_home/workspace/qywk/开发源码/Work/qywk-idea

mvn clean install

#tomcat位置
TOMCAT_PATH=/data/apache-tomcat-8.0.44

#服务名称
SERVER_NAME=qianYuWeiKe-0.0.1-SNAPSHOT
# 源jar路径,mvn打包完成之后，target目录下的jar包名称，也可选择成为war包，war包可移动到Tomcat的webapps目录下运行，这里使用jar包，用java -jar 命令执行  
JAR_NAME=qianYuWeiKe-0.0.1-SNAPSHOT
JAR_NWE_NAME=qianYuWeiKe

# 源jar路径  
#/usr/local/jenkins_home/workspace--->jenkins 工作目录
#demo 项目目录
#target 打包生成jar包的目录
JAR_PATH=/data/jenkins_home/workspace/qywk/开发源码/Work/qywk-idea/qianYuWeiKe/target
# 打包完成之后，把jar包移动到运行jar包的目录--->work_daemon，work_daemon这个目录需要自己提前创建
JAR_WORK_PATH=/data/run

echo "查询进程id-->$SERVER_NAME"
PID=`ps -ef | grep "$SERVER_NAME" | awk '{print $2}'`
echo "得到进程ID：$PID"
echo "结束进程"
for id in $PID
do
	kill -9 $id  
	echo "killed $id"  
done
echo "结束进程完成"

#复制jar包到执行目录
echo "复制jar包到执行目录:cp $JAR_PATH/$JAR_NAME.war $JAR_WORK_PATH"
cp $JAR_PATH/$JAR_NAME.war $JAR_WORK_PATH
echo "复制jar包完成"
cd $JAR_WORK_PATH
#修改文件权限
chmod 755 $JAR_NAME.war

chmod +x $TOMCAT_PATH/bin/*.sh 

cp $JAR_WORK_PATH/$JAR_NAME.war $TOMCAT_PATH/webapps

mv $TOMCAT_PATH/webapps/$JAR_NAME.war $TOMCAT_PATH/webapps/$JAR_NWE_NAME.war
 
#tomcat进程的id并kill掉
ps -ef |grep tomcat  |awk {'print $2'} | sed -e "s/^/kill -9 /g" | sh -
 
#删除tomcat日志文件
rm  $$TOMCAT_PATH/logs/* -rf
 
#删除tomcat的临时目录
rm  $$TOMCAT_PATH/work/* -rf

#停止tomcat
$TOMCAT_PATH/bin/shutdown.sh
 
#启动tomcat
$TOMCAT_PATH/bin/startup.sh 
 
#看启动日志
tail -f $TOMCAT_PATH/logs/catalina.out
```