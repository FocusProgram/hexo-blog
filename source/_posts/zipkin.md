---
title: zipkin
date: 2020-07-11 00:59:27
tags: 
 - SpringCloud 
 - 微服务
categories: 微服务
index_img: /img/default.png
---

**Zipkin链路追踪与监控**

---

# 1. 分布式链路监控与追踪产生背景

> 在微服务系统中，随着业务的发展，系统会变得越来越大，那么各个服务之间的调用关系也就变得越来越复杂。一个 HTTP 请求会调用多个不同的微服务来处理返回最后的结果，在这个调用过程中，可能会因为某个服务出现网络延迟过高或发送错误导致请求失败，这个时候，对请求调用的监控就显得尤为重要了。Spring Cloud Sleuth 提供了分布式服务链路监控的解决方案。

# 2. zipkin框架介绍

![](http://image.focusprogram.top/20200506154252.png)

> [zipkin](https://zipkin.io/)是 Twitter 的一个开源项目，它基于 Google Dapper 实现的。我们可以使用它来收集各个服务器上请求链路的跟踪数据，并通过它提供的 REST API 接口来辅助查询跟踪数据以实现对分布式系统的监控程序，从而及时发现系统中出现的延迟过高问题。除了面向开发的 API 接口之外，它还提供了方便的 UI 组件来帮助我们直观地搜索跟踪信息和分析请求链路明细，比如可以查询某段时间内各用户请求的处理时间等。
>
> Zipkin 和 Config 结构类似，分为服务端 Server，客户端 Client，客户端就是各个微服务应用。

# 3. zipkin部署

## 3.1 docker

以下两种部署方式均可：

```
$ docker run -d -p 9411:9411 openzipkin/zipkin
```

```
$ vim docker-compose.yml

version: '3'
services:
  web:
    image: openzipkin/zipkin
    hostname: zipkin
    container_name: zipkin
    ports:
     - 9400:9411
    environment:
      - STORAGE_TYPE=mysql
      - MYSQL_DB=zipkin
      - MYSQL_USER=root
      - MYSQL_PASS=root
      - MYSQL_HOST=114.55.34.44
      - MYSQL_TCP_PORT=3306
      - SET_CONTAINER_TIMEZONE=true
      - CONTAINER_TIMEZONE=Asia/Shanghai
     volumes:
      - /etc/localtime:/etc/localtime
      - /etc/timezone:/etc/timezone
    networks:
      default:
        ipv4_address: 172.18.0.11
    restart: always

networks:
  default:
    external:
      name: mynetwork
      
$ docker-compose up -d
```

## 3.2 jar包运行

```
$ curl -sSL https://zipkin.io/quickstart.sh | bash -s

$ java -jar zipkin.jar
```

## 3.3 源码编译

```
# get the latest source
$ git clone https://github.com/openzipkin/zipkin

$ cd zipkin

# Build the server and also make its dependencies
$ ./mvnw -DskipTests --also-make -pl zipkin-server clean install

# Run the server
$ java -jar ./zipkin-server/target/zipkin-server-*exec.jar
```

# 4. 项目构建

- **项目模块分为**
 - zipkin-server zipkin 服务监控
 - zipkin-consumer 服务消费者
 - zipkin-provider 服务提供者

## 4.1 zipkinp-server

引入maven依赖

```
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>1.5.2.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Camden.SR6</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>io.zipkin.java</groupId>
        <artifactId>zipkin-server</artifactId>
    </dependency>
    
    <dependency>
        <groupId>io.zipkin.java</groupId>
        <artifactId>zipkin-autoconfigure-ui</artifactId>
    </dependency>
</dependencies>
```

编辑配置文件application.yml

```
server:
  port: 9411
```

编辑启动主类ZipkinServerApplication

```
@SpringBootApplication
@EnableZipkinServer
public class ZipkinServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinServerApplication.class, args);
    }

}
```

## 4.2 zipkin-comsumber

引入maven依赖

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zipkin</artifactId>
    </dependency>
</dependencies>
```

编辑配置文件

```
server:
  port: 9100
spring:
  application:
    name: zipkin-consumer
  # 指定zipkin的地址
  zipkin:
    base-url: http://114.55.34.44:9400
    #base-url: http://localhost:9411
  # 默认sleuth收集信息的比率是0.1
  sleuth:
    sampler:
      percentage: 1
```

编辑启动主类ZipkinConsumerApplication

```
@SpringBootApplication
public class ZipkinConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinConsumerApplication.class, args);
    }

    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

//    指定sleuth收集比率，解决sleuth收集问题
//    @Bean
//    public AlwaysSampler defaultSampler(){
//        return new AlwaysSampler();
//    }

}
```

编辑ConsumerController

```
@RestController
public class ConsumerController {

    private static final Logger LOG = Logger.getLogger(ConsumerController.class.getName());

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("/consumer")
    public String callHome() {
        LOG.log(Level.INFO, "请求 service-consumer");
        LOG.log(Level.INFO, "远程调用：http://localhost:9200/provider");
        return restTemplate.getForObject("http://localhost:9200/provider", String.class);
    }

    @RequestMapping("/consumerInfo")
    public String info() {
        LOG.log(Level.INFO, "请求 service-consumer");
        return "i'm service-consumer";
    }

}
```

## 4.3 zipkin-provider

引入maven依赖

```
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zipkin</artifactId>
    </dependency>
</dependencies>
```

编辑配置文件

```
server:
  port: 9200
spring:
  application:
    name: zipkin-consumer
  # 指定zipkin的地址
  zipkin:
    base-url: http://114.55.34.44:9400
    #base-url: http://localhost:9411
  # 默认sleuth收集信息的比率是0.1
  sleuth:
    sampler:
      percentage: 1
```

编辑主启动类ZipkinProviderApplication

```
@SpringBootApplication
public class ZipkinProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinProviderApplication.class, args);
    }

    @Bean
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

}
```

编辑ProviderController

```
@RestController
public class ProviderController {

    private static final Logger LOG = Logger.getLogger(ProviderController.class.getName());

    @Autowired
    private RestTemplate restTemplate;

    @RequestMapping("/provider")
    public String callHome() {
        LOG.log(Level.INFO, "请求 service-provider");
        LOG.log(Level.INFO, "远程调用：http://localhost:9100/consumerInfo");
        return restTemplate.getForObject("http://localhost:9100/consumerInfo", String.class);
    }

    @RequestMapping("/providerInfo")
    public String info() {
        LOG.log(Level.INFO, "请求 service-provider ");
        return "i'm service-provider";
    }

}
```

## 4.4 启动部署

分别依次启动三个项目，然后访问

```
$ curl http://localhost:9100/consumer
```

![](http://image.focusprogram.top/20200506160018.png)

访问zipkin服务监控地址

[https://www.focusprogram.top/zipkin](https://www.focusprogram.top/zipkin)

![](http://image.focusprogram.top/20200506160218.png)

![](http://image.focusprogram.top/20200506160257.png)