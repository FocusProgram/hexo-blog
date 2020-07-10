---
title: Nacos
date: 2020-07-11 01:53:00
tags: 
 - SpringCloud 
 - 微服务
categories: 微服务
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/nacosMap.jpg
---

**Sping Cloud Alibaba Nacos**

---

# 1. 什么是Nacos？

> [Nacos](https://github.com/alibaba/Nacos) 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

# 2. Nacos的关键特性

- 服务发现和服务健康监测

> Nacos 支持基于 DNS 和基于 RPC 的服务发现。服务提供者使用 原生SDK、OpenAPI、或一个独立的Agent TODO注册 Service 后，服务消费者可以使用DNS TODO 或HTTP&API查找和发现服务。
>
> Nacos 提供对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求。Nacos 支持传输层 (PING 或 TCP)和应用层 (如 HTTP、MySQL、用户自定义）的健康检查。 对于复杂的云环境和网络拓扑环境中（如 VPC、边缘网络等）服务的健康检查，Nacos 提供了 agent 上报模式和服务端主动检测2种健康检查模式。Nacos 还提供了统一的健康检查仪表盘，帮助您根据健康状态管理服务的可用性及流量。

- 动态配置服务

> 动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。
>
> 动态配置消除了配置变更时重新部署应用和服务的需要，让配置管理变得更加高效和敏捷。
>
> 配置中心化管理让实现无状态服务变得更简单，让服务按需弹性扩展变得更容易。
>
> Nacos 提供了一个简洁易用的UI [控制台样例](http://console.nacos.io/nacos/index.html) 帮助您管理所有的服务和应用的配置。Nacos 还提供包括配置版本跟踪、金丝雀发布、一键回滚配置以及客户端配置更新状态跟踪在内的一系列开箱即用的配置管理特性，帮助您更安全地在生产环境中管理配置变更和降低配置变更带来的风险。

- 动态 DNS 服务

> 动态 DNS 服务支持权重路由，让您更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务。动态DNS服务还能让您更容易地实现以 DNS 协议为基础的服务发现，以帮助您消除耦合到厂商私有服务发现 API 上的风险。
>
> Nacos 提供了一些简单的 DNS APIs TODO 帮助您管理服务的关联域名和可用的 IP:PORT 列表.

- 服务及其元数据管理

> Nacos 能让您从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA 以及最首要的 metrics 统计数据。

# 3. Nacos生态

![](https://gitee.com/FocusProgram/PicGo/raw/master/nacosMap.jpg)

![](https://gitee.com/FocusProgram/PicGo/raw/master/1533045871534-e64b8031-008c-4dfc-b6e8-12a597a003fb.png)

# 4. 启动 Nacos Server 并添加配置

## 4.1 [下载地址](https://github.com/alibaba/nacos/releases)

## 4.2 启动Nacos Service

```
Linux/Unix/Mac 操作系统，执行命令(默认集群版本启动):

`sh startup.sh -m standalone`(单机) `sh startup.sh -m cluster`(集群)

Windows 操作系统，执行命令 `cmd startup.cmd`

默认账户密码：nacos/naocs
```

## 4.3 命令添加配置

```
curl -X POST "http://127.0.0.1:8848/nacos/v1/cs/configs?dataId=nacosConfig.properties&group=DEFAULT_GROUP&content=user.name=Mr.Kong%0Auser.age=24"
```

```
添加配置如下：
dataId 为 nacosConfig.properties
group 为 DEFAULT_GROUP

内容如下:
user.name=Mr.Kong
user.age=24
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200305152113.png)

# 5. Nacos Config 整合


## 5.1 引入maven依赖

```
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
 </dependency>
```

## 5.2 添加bootstrap.properties

```
spring.application.name=nacosConfig  #保持跟Nacos的id名一致
spring.cloud.nacos.config.server-addr=ip:8848 #服务器ip或本地ip
```

## 5.3 添加application.properties

```
server.port=9000
```

## 5.4 获取相应的配置

```java
@RefreshScope //打开动态刷新功能
 class SampleController {

 	@Value("${user.name}")
 	String userName;

 	@Value("${user.age}")
 	int age;
 }
```

## 5.5 应用启动

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200305153453.png)


## 5.6 原理解析

### 5.6.1 Nacos Config 数据结构

```
Nacos Config 主要通过 dataId 和 group 来唯一确定一条配置

Nacos Client 从 Nacos Server 端获取数据时，调用的是此接口
ConfigService.getConfig(String dataId, String group, long timeoutMs)
```

### 5.6.2 Spring Cloud 应用获取数据

#### 5.6.2.1 dataId

```
在 Nacos Config Starter 中，dataId 的拼接格式如下

${prefix} - ${spring.profiles.active} . ${file-extension}

- `prefix` 默认为 `spring.application.name` 的值，也可以通过配置项 `spring.cloud.nacos.config.prefix`来配置。

- `spring.profiles.active` 即为当前环境对应的 profile，详情可以参考 [Spring Boot文档](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-profiles.html#boot-features-profiles)

-  注意:当 activeprofile 为空时，对应的连接符 - 也将不存在，dataId 的拼接格式变成 prefix.prefix

- `file-extension` 为配置内容的数据格式，可以通过配置项 `spring.cloud.nacos.config.file-extension`来配置。 目前只支持 `properties` 类型。
```

#### 5.6.2.2 group

```
- `group` 默认为 `DEFAULT_GROUP`，可以通过 `spring.cloud.nacos.config.group` 配置。
```

#### 5.6.2.3 自动注入

```
Nacos Config Starter 实现了 `org.springframework.cloud.bootstrap.config.PropertySourceLocator`接口，并将优先级设置成了最高。

在 Spring Cloud 应用启动阶段，会主动从 Nacos Server 端获取对应的数据，并将获取到的数据转换成 PropertySource 且注入到 Environment 的 PropertySources 属性中，所以使用 @Value 注解也能直接获取 Nacos Server 端配置的内容。
```

#### 5.6.2.4 动态刷新

```
Nacos Config Starter 默认为所有获取数据成功的 Nacos 的配置项添加了监听功能，在监听到服务端配置发生变化时会实时触发 `org.springframework.cloud.context.refresh.ContextRefresher` 的 refresh 方法 。

如果需要对 Bean 进行动态刷新，给类添加 `@RefreshScope` 或 `@ConfigurationProperties`注解。
```

#### 5.6.2.5 Endpoint 信息查看

```
Springboot支持这一点，Nacos Config也同时可以使用Endpoint来暴露信息。

在maven 中添加 `spring-boot-starter-actuator`依赖，并在配置中允许 Endpoints 的访问。

- Spring Boot 1.x 中添加配置 management.security.enabled=false
- Spring Boot 2.x 中添加配置 management.endpoints.web.exposure.include=*

Spring Boot 1.x 可以通过访问 [http://127.0.0.1:9000/nacos_config](http://127.0.0.1:9000/nacos_config) 来查看 Nacos Endpoint 的信息。

Spring Boot 2.x 可以通过访问 [http://127.0.0.1:9000/actuator/nacos-config](http://127.0.0.1:9000/actuator/nacos-config) 来访问。
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200305162410.png)


## 5.7 更多

| 配置项             | key                                            | 默认值            | 说明                                       |
|-----------------|------------------------------------------------|----------------|------------------------------------------|
| 服务端地址           | spring\.cloud\.nacos\.config\.server\-addr     |                |                                          |
| DataId前缀        | spring\.cloud\.nacos\.config\.prefix           |                | spring\.application\.name                |
| Group           | spring\.cloud\.nacos\.config\.group            | DEFAULT\_GROUP |                                          |
| dataID后缀及内容文件格式 | spring\.cloud\.nacos\.config\.file\-extension  | properties     | dataId的后缀，同时也是配置内容的文件格式，目前只支持 properties |
| 配置内容的编码方式       | spring\.cloud\.nacos\.config\.encode           | UTF\-8         | 配置的编码                                    |
| 获取配置的超时时间       | spring\.cloud\.nacos\.config\.timeout          | 3000           | 单位为 ms                                   |
| 配置的命名空间         | spring\.cloud\.nacos\.config\.namespace        |                | 常用场景之一是不同环境的配置的区分隔离，例如开发测试环境和生产环境的资源隔离等。 |
| AccessKey       | spring\.cloud\.nacos\.config\.access\-key      |                |                                          |
| SecretKey       | spring\.cloud\.nacos\.config\.secret\-key      |                |                                          |
| 相对路径            | spring\.cloud\.nacos\.config\.context\-path    |                | 服务端 API 的相对路径                            |
| 接入点             | spring\.cloud\.nacos\.config\.endpoint         | UTF\-8         | 地域的某个服务的入口域名，通过此域名可以动态地拿到服务端地址           |
| 是否开启监听和自动刷新     | spring\.cloud\.nacos\.config\.refresh\.enabled | TRUE           |                                          |


# 6. Nacos Discovery 整合

> Nacos Discovery Starter 完成 Spring Cloud 应用的服务注册与发现。

## 6.1 引入maven依赖

```
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
 </dependency>

```

## 6.2 添加application.properties

```
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
```

## 6.3 服务注册和发现

```
@SpringBootApplication
@EnableDiscoveryClient
public class ProviderApplication {

 	public static void main(String[] args) {
 		SpringApplication.run(Application.class, args);
 	}

 	@RestController
 	class EchoController {
 		@RequestMapping(value = "/echo/{string}", method = RequestMethod.GET)
 		public String echo(@PathVariable String string) {
 				return string;
 		}
 	}
 }
```

## 6.4 查询服务

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200305163858.png)

## 6.5 服务发现

```
Nacos Discovery Starter默认集成了Ribbon，因此对于使用Ribbon进行负载平衡的组件，您> 可以直接使用Nacos服务发现。

NacosServerList实现com.netflix.loadbalancer.ServerList接口并在@ConditionOnMissing> Bean下自动注入它。
```

## 6.6 原理

```
服务注册：

在Spring云应用程序的启动阶段，将监视WebServerInitializedEvent事件。在初始化Web容器后收到WebServerInitializedEvent事件时，将触发注册操作，并调用ServiceRegistry注册方法以将服务注册到Nacos Server。

服务发现：

NacosServerList实现com.netflix.loadbalancer.ServerList接口并在@ConditionOnMissingBean下自动注入它。功能区默认为集成。
```

## 6.7 更多

| 配置项       | 键                                                   | 默认值                       | 描述                                                  |
|-----------|-----------------------------------------------------|---------------------------|-----------------------------------------------------|
| 服务器地址     | spring\.cloud\.nacos\.discovery\.server\-addr       |                           | nacos注册中心地址                                         |
| 服务名       | spring\.cloud\.nacos\.discovery\.service            | spring\.application\.name | 服务名                                                 |
| 权重        | spring\.cloud\.nacos\.discovery\.weight             | 1                         | 值从1到100，值越大，重量越大                                    |
| IP        | spring\.cloud\.nacos\.discovery\.ip                 |                           | ip address to registry，最高优先级                        |
| 网络接口      | spring\.cloud\.nacos\.discovery\.network\-interface |                           | 未配置IP时，注册的IP地址为网络接口对应的IP地址。如果未配置此项，则默认采用第一个网络接口的地址。 |
| 端口        | spring\.cloud\.nacos\.discovery\.port               | \-1                       | 注册端口，无需配置即可自动检测                                     |
| namesapce | spring\.cloud\.nacos\.discovery\.namespace          |                           | 开发环境（dev、pro等）                                      |
| accesskey | spring\.cloud\.nacos\.discovery\.access\-key        |                           |                                                     |
| secretkey | spring\.cloud\.nacos\.discovery\.secret\-key        |                           |                                                     |
| 元数据       | spring\.cloud\.nacos\.discovery\.metadata           |                           | 扩展数据，使用Map格式配置                                      |
| 日志名称      | spring\.cloud\.nacos\.discovery\.log\-name          |                           |                                                     |
| 端点        | spring\.cloud\.nacos\.discovery\.endpoint           |                           | 服务的域名，通过该域名可以动态获取服务器地址。                             |
| 集成功能区     | ribbon\.nacos\.enabled                              | TRUE                      |                                                     |