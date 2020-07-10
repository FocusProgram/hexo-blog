---
title: Apollo
date: 2020-07-11 01:52:34
tags: 
 - SpringCloud 
 - 微服务
categories: 微服务
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/basic-architecture.png
---

**Apollo**

---

# 1. 什么是Apollo?

[Apollo](https://github.com/ctripcorp/apollo)
（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

# 2. 特点

* **统一管理不同环境、不同集群的配置**
  * Apollo提供了一个统一界面集中式管理不同环境（environment）、不同集群（cluster）、不同命名空间（namespace）的配置。
  * 同一份代码部署在不同的集群，可以有不同的配置，比如zk的地址等
  * 通过命名空间（namespace）可以很方便的支持多个不同应用共享同一份配置，同时还允许应用对共享的配置进行覆盖
  * 配置界面支持多语言（中文，English）

* **配置修改实时生效（热发布）**
  * 用户在Apollo修改完配置并发布后，客户端能实时（1秒）接收到最新的配置，并通知到应用程序。

* **版本发布管理**
  * 所有的配置发布都有版本概念，从而可以方便的支持配置的回滚。

* **灰度发布**
  * 支持配置的灰度发布，比如点了发布后，只对部分应用实例生效，等观察一段时间没问题后再推给所有应用实例。

* **权限管理、发布审核、操作审计**
  * 应用和配置的管理都有完善的权限管理机制，对配置的管理还分为了编辑和发布两个环节，从而减少人为的错误。
  * 所有的操作都有审计日志，可以方便的追踪问题。

* **客户端配置信息监控**
  * 可以方便的看到配置在被哪些实例使用

* **提供Java和.Net原生客户端**
  * 提供了Java和.Net的原生客户端，方便应用集成
  * 支持Spring Placeholder，Annotation和Spring Boot的ConfigurationProperties，方便应用使用（需要Spring 3.1.1+）
  * 同时提供了Http接口，非Java和.Net应用也可以方便的使用

* **提供开放平台API**
  * Apollo自身提供了比较完善的统一配置管理界面，支持多环境、多数据中心配置管理、权限、流程治理等特性。
  * 不过Apollo出于通用性考虑，对配置的修改不会做过多限制，只要符合基本的格式就能够保存。
  * 在我们的调研中发现，对于有些使用方，它们的配置可能会有比较复杂的格式，如xml, json，需要对格式做校验。
  * 还有一些使用方如DAL，不仅有特定的格式，而且对输入的值也需要进行校验后方可保存，如检查数据库、用户名和密码是否匹配。
  * 对于这类应用，Apollo支持应用方通过开放接口在Apollo进行配置的修改和发布，并且具备完善的授权和权限控制

# 3. 设计([官方文档参考地址](https://github.com/ctripcorp/apollo/wiki/Apollo%E9%85%8D%E7%BD%AE%E4%B8%AD%E5%BF%83%E4%BB%8B%E7%BB%8D))

## 3.1 基础模型

如下即是Apollo的基础模型：

1. 用户在配置中心对配置进行修改并发布
2. 配置中心通知Apollo客户端有配置更新
3. Apollo客户端从配置中心拉取最新的配置、更新本地配置并通知到应用

![](https://gitee.com/FocusProgram/PicGo/raw/master/basic-architecture.png)

## 3.2 界面概览

![](https://gitee.com/FocusProgram/PicGo/raw/master/apollo-home-screenshot.png)

## 3.3  添加/修改配置项

用户可以通过配置中心界面方便的添加/修改配置项，更多使用说明请参见应用[接入指南](https://github.com/ctripcorp/apollo/wiki/%E5%BA%94%E7%94%A8%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97)

![](https://gitee.com/FocusProgram/PicGo/raw/master/edit-item-entry.png)

输入配置信息：

![](https://gitee.com/FocusProgram/PicGo/raw/master/edit-item.png)

## 3.4  发布配置

通过配置中心发布配置：

![](https://gitee.com/FocusProgram/PicGo/raw/master/publish-items-entry.png)

填写发布信息：

![](https://gitee.com/FocusProgram/PicGo/raw/master/publish-items.png)


## 3.5 客户端获取配置（Java API样例）

配置发布后，就能在客户端获取到了，以Java为例，获取配置的示例代码如下。Apollo客户端还支持和Spring整合，更多客户端使用说明请参见[Java客户端使用指南](https://github.com/ctripcorp/apollo/wiki/Java%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97)。

```java
Config config = ConfigService.getAppConfig();
Integer defaultRequestTimeout = 200;
Integer requestTimeout = config.getIntProperty("requestTimeout", defaultRequestTimeout);
```

## 3.6 客户端监听配置变化

通过上述获取配置代码，应用就能实时获取到最新的配置了。

不过在某些场景下，应用还需要在配置变化时获得通知，比如数据库连接的切换等，所以Apollo还提供了监听配置变化的功能，Java示例如下：

```java
Config config = ConfigService.getAppConfig();
config.addChangeListener(new ConfigChangeListener() {
  @Override
  public void onChange(ConfigChangeEvent changeEvent) {
    for (String key : changeEvent.changedKeys()) {
      ConfigChange change = changeEvent.getChange(key);
      System.out.println(String.format(
        "Found change - key: %s, oldValue: %s, newValue: %s, changeType: %s",
        change.getPropertyName(), change.getOldValue(),
        change.getNewValue(), change.getChangeType()));
     }
  }
});
```

## 3.7 Spring集成样例

Apollo和Spring也可以很方便地集成，只需要标注@EnableApolloConfig后就可以通过@Value获取配置信息：

```java
@Configuration
@EnableApolloConfig
public class AppConfig {}
@Component
public class SomeBean {
    //timeout的值会自动更新
    @Value("${request.timeout:200}")
    private int timeout;
}
```

# 4. 客户端集成Apollo

## 4.1 开发环境

### 4.1.1 java

* **Apollo服务端：1.8+**
* **Apollo客户端：1.7+**

由于Quick Start会在本地同时启动服务端和客户端，所以需要在本地安装Java 1.8+。

在配置好后，可以通过如下命令检查：

```linux
java -version
样例输出：

java version "1.8.0_74"
Java(TM) SE Runtime Environment (build 1.8.0_74-b02)
Java HotSpot(TM) 64-Bit Server VM (build 25.74-b02, mixed mode)
Windows用户请确保JAVA_HOME环境变量已经设置。
```


### 4.1.2 mysql

* **版本要求：5.6.5+**

Apollo的表结构对timestamp使用了多个default声明，所以需要5.6.5以上版本。

连接上MySQL后，可以通过如下命令检查：

```linux
SHOW VARIABLES WHERE Variable_name = 'version';
```

### 4.1.3 下载Quick Start安装包

我们准备好了一个Quick Start安装包，大家只需要下载到本地，就可以直接使用，免去了编译、打包过程。

安装包共50M，如果访问github网速不给力的话，可以从百度网盘下载。

* **从Github下载**
  * checkout或下载[apollo-build-scripts](https://github.com/nobodyiam/apollo-build-scripts)项目
  * 由于Quick Start项目比较大，所以放在了另外的repository，请注意[项目地址](https://github.com/nobodyiam/apollo-build-scripts)
https://github.com/nobodyiam/apollo-build-scripts

```linux
$ git clone https://github.com/nobodyiam/apollo-build-scripts.git
```

* **从百度网盘下载**

  * [百度网盘下载地址](https://pan.baidu.com/s/1mhVf9va#list/path=/sharelink1426331153-165614845139829/apollo-quick-start&parentPath=/sharelink1426331153-165614845139829)

Quick Start只针对本地测试使用，所以一般用户不需要自己下载源码打包，只需要下载已经打好的包即可。不过也有部分用户希望在修改代码后重新打包，那么可以参考如下步骤：

修改apollo-configservice, apollo-adminservice和apollo-portal的pom.xml，注释掉spring-boot-maven-plugin和maven-assembly-plugin
在根目录下执行mvn clean package -pl apollo-assembly -am -DskipTests=true
复制apollo-assembly/target下的jar包，rename为apollo-all-in-one.jar

## 4.2 安装部署

### 4.2.1 创建数据库

Apollo服务端共需要两个数据库：ApolloPortalDB和ApolloConfigDB，我们把数据库、表的创建和样例数据都分别准备了sql文件，只需要导入数据库即可。

> 注意：如果你本地已经创建过Apollo数据库，请注意备份数据。我们准备的sql文件会清空Apollo相关的表。

执行以下sql导入sql执行脚本：

```sql
source /your_local_path/sql/apolloportaldb.sql

source /your_local_path/sql/apolloconfigdb.sql
```

### 4.2.2 配置数据库链接

Apollo服务端需要知道如何连接到你前面创建的数据库，所以需要编辑demo.sh，修改ApolloPortalDB和ApolloConfigDB相关的数据库连接串信息。

> 注意：填入的用户需要具备对ApolloPortalDB和ApolloConfigDB数据的读写权限。

```linux
#apollo config db info
apollo_config_db_url=jdbc:mysql://localhost:3306/ApolloConfigDB?characterEncoding=utf8
apollo_config_db_username=用户名
apollo_config_db_password=密码（如果没有密码，留空即可）

# apollo portal db info
apollo_portal_db_url=jdbc:mysql://localhost:3306/ApolloPortalDB?characterEncoding=utf8
apollo_portal_db_username=用户名
apollo_portal_db_password=密码（如果没有密码，留空即可）
```

### 4.2.3 启动Apollo

Quick Start脚本会在本地启动3个服务，分别使用8070, 8080, 8090端口，请确保这3个端口当前没有被使用。

例如，在Linux/Mac下，可以通过如下命令检查：

```linux
lsof -i:8080 查看8080端口是否被占用

netstart -tunlp|grep 8080 查看8080端口是否被占用

ps aux|grep apollo 查看Apollo进程是否运行
```

执行脚本:

```linux
./demo.sh start  开启服务端运行（start services and portal）
./demo.sh stop   停止服务端运行（stop services and portal）
./demo.sh client 开启客户端运行（start client demo program）
```

输出如下内容，说明启动成功：

```linux
==== starting service ====
Service logging file is ./service/apollo-service.log
Started [10768]
Waiting for config service startup.......
Config service started. You may visit http://localhost:8080 for service status now!
Waiting for admin service startup....
Admin service started
==== starting portal ====
Portal logging file is ./portal/apollo-portal.log
Started [10846]
Waiting for portal startup......
Portal started. You can visit http://localhost:8070 now!
```

## 4.3 使用Apollo配置中心

### 4.3.1 服务器端搭建

访问[http://localhost:8070](http://localhost:8070)

![](https://gitee.com/FocusProgram/PicGo/raw/master/apollo-login.png)

默认用户名密码apollo/admin,如需修改参考[Portal 实现用户登录功能
](https://github.com/ctripcorp/apollo/wiki/Portal-%E5%AE%9E%E7%8E%B0%E7%94%A8%E6%88%B7%E7%99%BB%E5%BD%95%E5%8A%9F%E8%83%BD)

![](https://gitee.com/FocusProgram/PicGo/raw/master/apollo-sample-home.png)

点击SampleApp进入配置界面，可以看到当前有一个配置timeout=100

![](https://gitee.com/FocusProgram/PicGo/raw/master/sample-app-config.png)

> 如果提示系统出错，请重试或联系系统负责人，请稍后几秒钟重试一下，因为通过Eureka注册的服务有一个刷新的延时。

新建一个配置中心:

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307105252.png)

新增配置：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307105521.png)

> 注：可以通过文本的方式一次添加多个配置

点击发布：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307105824.png)

### 4.3.2 客户端搭建基于SpringBoot

1.引入maven依赖

```java
<!-- apollo 携程apollo配置中心框架 -->
        <dependency>
            <groupId>com.ctrip.framework.apollo</groupId>
            <artifactId>apollo-client</artifactId>
            <version>1.0.0</version>
        </dependency>
        <dependency>
            <groupId>com.ctrip.framework.apollo</groupId>
            <artifactId>apollo-core</artifactId>
            <version>1.0.0</version>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
```
2.项目结构为：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307104203.png)

* **app.properties**
```
app.id=apolloConfig
apollo.meta=http://114.55.34.44:8080
```

* **apollo-env.properties**
```
local.meta=http://114.55.34.44:8080
dev.meta=http://114.55.34.44:8080
fat.meta=${fat_meta}
uat.meta=${uat_meta}
lpt.meta=${lpt_meta}
pro.meta=${pro_meta}

```

* **application.yml**
```
server:
  port: 9000
spring:
  application:
    name: apollo-demo
eureka:
  client:
    service-url:
      defaultZone: http://114.55.34.44:8080/eureka
```

Controller层实现：
```
@RestController
@RequestMapping("apollo")
public class ApolloController {

    @Value("${name:无法读取到值}")
    private String name;

    @Value("${age:0}")
    private Long age;

    @RequestMapping("/getname")
    public String getName() {
        return name + "的年龄" + age;
    }
}
```

项目启动类代码类实现：
```
@Configuration
@SpringBootApplication
@EnableApolloConfig
public class ApollodemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(ApollodemoApplication.class, args);
    }

}
```

访问 [http://localhost:9000/apollo/getname](http://localhost:9000/apollo/getname)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307111511.png)

修改name值后，重新访问，实时动态更新

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307111635.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307111710.png)

## 4.4 使用Apollo灰度发布

### 4.4.1 准备两台服务器

| 集群节点      | 端口   |
|-------------------|------|
| 192\.168\.80\.128 | 9000 |
| 192\.168\.80\.135 | 9000 |

访问[http://192.168.80.128:9000/apollo/getname](http://192.168.80.128:9000/apollo/getname)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112459.png)

访问[http://192.168.80.135:9000/apollo/getname](http://192.168.80.135:9000/apollo/getname)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112514.png)

Apollo添加灰度发布

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112720.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112758.png)

针对192.168.80.128节点进行灰度发布

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112850.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112917.png)

访问[http://192.168.80.128:9000/apollo/getname](http://192.168.80.128:9000/apollo/getname)已经发生变化

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112940.png)

访问[http://192.168.80.135:9000/apollo/getname](http://192.168.80.135:9000/apollo/getname)未发生变化

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307112514.png)

> 至此，Apollo的灰度发布已经生效，如果测试灰度发布节点没有问题可以进行全量发布同步到所有的节点，如果测试灰度发布节点存在问题可以放弃灰度发布，会恢复至以前的配置文件版本。

# 5. 常见问题

* **问题一：只在Maven中导入了client依赖**

```
Exception in thread "main" java.lang.NoClassDefFoundError: Could not initialize class com.ctrip.framework.apollo.tracer.Tracer
      at com.ctrip.framework.apollo.build.ApolloInjector.getInstance(ApolloInjector.java:37)
      at com.ctrip.framework.apollo.ConfigService.getManager(ConfigService.java:25)
      at com.ctrip.framework.apollo.ConfigService.getConfig(ConfigService.java:61)
```

解决办法：

导入同版本的apollo-core依赖

* **问题二：eureka解析为内网地址**

```
Sync config failed, will retry. Repository class com.ctrip.framework.apollo.internals.RemoteConfigRepository, 
reason: Load Apollo Config failed - appId: bitongchong_bos, cluster: default, namespace: application, 
url: http://此处是私有ip:8080/configs/bitongchong_bos/default/application?ip=192.168.102.1&messages=%7B%22details%22%3A%7B%22bitongchong_bos%2Bdefault%2Bapplication%22%3A6%7D%7D&releaseKey=20190803112627-2b5dd0e414976d16 
[Cause: Could not complete get operation [Cause: connect timed out]]
```

解决办法：

1. 项目启动时指定启动参数：

-Deureka.instance.ip-address=公网地址

2. 修改Apollo启动脚本：

-Deureka.instance.ip-address=114.55.34.44"

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307114400.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200307114508.png)

* **问题三：未对apollo.meta 属性进行正确赋值**

```
2020-03-04 01:34:28.169  WARN 21840 --- [           main] c.c.f.a.i.AbstractConfigRepository       : Sync config failed, will retry. Repository class com.ctrip.framework.apollo.internals.RemoteConfigRepository, reason: Get config services failed from http://xx.xx.xx.xx:8070/services/config?appId=SPOC_Platform&ip=192.168.102.1 [Cause: Could not complete get operation [Cause: java.lang.IllegalStateException: Expected BEGIN_ARRAY but was STRING at line 1 column 1 path $ [Cause: Expected BEGIN_ARRAY but was STRING at line 1 column 1 path $]]]
2020-03-04 01:34:30.511  WARN 21840 --- [           main] c.c.f.a.i.LocalFileConfigRepository      : Sync config from upstream repository class com.ctrip.framework.apollo.internals.RemoteConfigRepository failed, reason: Get config services failed from http://xx.xx.xx.xx:8070/services/config?appId=SPOC_Platform&ip=192.168.102.1 [Cause: Could not complete get operation [Cause: java.lang.IllegalStateException: Expected BEGIN_ARRAY but was STRING at line 1 column 1 path $ [Cause: Expected BEGIN_ARRAY but was STRING at line 1 column 1 path $]]]
```

解决办法：

这个是未对apollo.meta 属性进行正确赋值，必须是eureka的地址和端口，就是你输入这个url+端口的时候跳转到的是eureka这个界面，而非apollo管理页面，否则必定报错

```
-Dapollo.meta=http://ip:8080 -Denv=dev
```

# 6. Apollo相比于Spring Cloud Config有什么优势?

| 功能点      | Apollo                                                      | Spring Cloud Config                  | 备注                                                   |
|----------|-------------------------------------------------------------|--------------------------------------|------------------------------------------------------|
| 配置界面     | 一个界面管理不同环境、不同集群配置                                           | 无，需要通过git操作                          |                                                      |
| 配置生效时间   | 实时                                                          | 重启生效，或手动refresh生效                    | Spring Cloud Config需要通过Git webhook，加上额外的消息队列才能支持实时生效 |
| 版本管理     | 界面上直接提供发布历史和回滚按钮                                            | 无，需要通过git操作                          |                                                      |
| 灰度发布     | 支持                                                          | 不支持                                  |                                                      |
| 授权、审核、审计 | 界面上直接支持，而且支持修改、发布权限分离                                       | 需要通过git仓库设置，且不支持修改、发布权限分离            |                                                      |
| 实例配置监控   | 可以方便的看到当前哪些客户端在使用哪些配置                                       | 不支持                                  |                                                      |
| 配置获取性能   | 快，通过数据库访问，还有缓存支持                                            | 较慢，需要从git clone repository，然后从文件系统读取 |                                                      |
| 客户端支持    | 原生支持所有Java和\.Net应用，提供API支持其它语言应用，同时也支持Spring annotation获取配置 | 支持Spring应用，提供annotation获取配置          | Apollo的适用范围更广一些                                      |
