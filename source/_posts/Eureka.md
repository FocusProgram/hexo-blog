---
title: Eureka
date: 2020-07-11 01:52:53
tags: 
 - SpringCloud 
 - 微服务
categories: 微服务
index_img: https://gitee.com/FocusProgram/PicGo/raw/master/20200310094547.png
---

**Eureka**

---

# 1. 什么是Eureka?

> [Eureka](https://github.com/Netflix/eureka) 是一个基于 REST (REST)的服务，主要用于 AWS 云中定位服务，用于中间层服务器的负载平衡和故障转移。 我们称这个服务为 Eureka 服务器。 Eureka 还带有一个基于 java 的客户机组件—— Eureka Client，它使得与服务的交互更加容易。 客户机还有一个内置的负载平衡器，可以进行基本的循环负载平衡。 在 Netflix，一个更加复杂的负载平衡器包装了 Eureka，以提供基于流量、资源使用、错误条件等几个因素的加权负载平衡，从而提供更好的弹性。

# 2. Eureka的组成?

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310094547.png)

- Eureka Server:提供服务注册和发现（响应失效剔除、自我保护）；

- Service Provider：服务提供方将自身服务注册到Eureka,从而使服务消费方能够找到（向注册中心服务注册、服务同步、服务续约）；

- Service Consumer服务消费方从Eureka获取注册服务列表，从而能够消费服务（往注册中心获取服务、服务调用、服务下线）

> 为了让Eureka Server知道客户端是否还活着，引入了心跳机制，即每隔一定时间（默认30s）都会告知Eureka Server我还活着，防止“剔除任务”将服务实例从服务列表中排除出去。如果发现一个服务死了，Eureka不会将其注册信息直接删除而是尽可能当前实例的注册信息保护起来即进入自我保护阶段（默认自我保护为开启，可以通过eureka.server.enable-self-preservation=false关闭自我保护）。SpringCloud的一些其它模块（比如Ribbon、Zuul等）可以通过Eureka Server来发现系统中其他微服务，做相关的逻辑处理。

# 3. SpringCloud集成Eureka

参考地址 [https://github.com/FocusProgram/springcloud-eureka](https://github.com/FocusProgram/springcloud-eureka)

## 3.1 模块

| Sever Name                         | Port | Function    |
|------------------------------------|------|-------------|
| springcloud\-eureka\-server\-one   | 9001 | 服务端1        |
| springcloud\-eureka\-server\-two   | 9002 | 服务端2        |
| springcloud\-eureka\-provider\-one | 8001 | 客户端1（服务提供者） |
| springcloud\-eureka\-provider\-two | 8002 | 客户端2（服务提供者） |
| springcloud\-eureka\-consumer      | 7000 | 客户端（服务消费者）  |

## 3.2 功能模块配置

### 3.2.1 eureka-server配置

1. **springcloud-eureka-server-one**和**springcloud-eureka-server-two**引入依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

2. **springcloud-eureka-server-one**配置文件

```
eureka:
  client:
    fetchRegistry: false #由于服务本身就为注册中心，因此不需要注册自己，所有设置为false
    registerWithEureka: false #由于注册中心的职责就是维护服务实例，不需要去检索服务，所有设置为false
    service-url:
      defaultZone: http://localhost:9002/eureka #服务注册中心的配置内容，指定服务注册中心的位置
  instance:
    hostname: eureka-server-one
server:
  port: 9001
spring:
  application:
    name: eureka-server-one
```

3. **springcloud-eureka-server-two**配置文件

```
eureka:
  client:
    fetchRegistry: false #由于服务本身就为注册中心，因此不需要注册自己，所有设置为false
    registerWithEureka: false #由于注册中心的职责就是维护服务实例，不需要去检索服务，所有设置为false
    service-url:
      defaultZone: http://localhost:9001/eureka #服务注册中心的配置内容，指定服务注册中心的位置
  instance:
    hostname: eureka-server-two
server:
  port: 9002
spring:
  application:
    name: eureka-server-two
```

4. 启动类添加@EnableEurekaServer注解

```
@SpringBootApplication
@EnableEurekaServer
public class SpringcloudEurekaServerOneApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringcloudEurekaServerOneApplication.class, args);
    }

}
```

```
@SpringBootApplication
@EnableEurekaServer
public class SpringcloudEurekaServerTwoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringcloudEurekaServerTwoApplication.class, args);
    }

}
```

5. 成功启动显示如下：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310144122.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310144134.png)

### 3.2.2 eureka-provider配置

1. **springcloud-eureka-provider-one**和**springcloud-eureka-provider-two**引入依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

2. **springcloud-eureka-provider-one**配置文件

```
eureka:
  client:
    fetchRegistry: true #注册到别的服务，所有设置为true
    registerWithEureka: true #需要去检索服务，所有设置为true
    service-url:
      defaultZone: http://localhost:9001/eureka,http://localhost:9002/eureka #服务注册中心的配置内容，指定服务注册中心的位置
#  instance:
#    hostname: eureka-provider
#    prefer-ip-address: true #以ip地址注册到服务中心（单节点部署为分布式Eureka集群，禁止设置为true）
  server:
    enable-self-preservation: false #是否开启自我保护模式
    eviction-interval-timer-in-ms: 60000 #服务注册表清理间隔（单位：毫秒，默认60*1000）
server:
  port: 8001
spring:
  application:
    name: eureka-provider
```

3. **springcloud-eureka-provider-two**配置文件

```
eureka:
  client:
    fetchRegistry: true #注册到别的服务，所有设置为true
    registerWithEureka: true #需要去检索服务，所有设置为true
    service-url:
      defaultZone: http://localhost:9001/eureka,http://localhost:9002/eureka #服务注册中心的配置内容，指定服务注册中心的位置
#  instance:
#    hostname: eureka-provider
#    prefer-ip-address: true #以ip地址注册到服务中心（单节点部署为分布式Eureka集群，禁止设置为true）
  server:
    enable-self-preservation: false #是否开启自我保护模式
    eviction-interval-timer-in-ms: 60000 #服务注册表清理间隔（单位：毫秒，默认60*1000）
server:
  port: 8002
spring:
  application:
    name: eureka-provider
```

4. 添加Controller访问层

```
@RestController
@RequestMapping("provider")
public class ProviderController {

    @Value("${server.port}")
    private String port;

    @GetMapping("/get")
    public String get() {
        System.out.println("------------->我是提供服务提供者,端口号为" + port);
        return "我是提供服务提供者,端口号为" + port;
    }
}
```

5. 启动类添加@EnableEurekaClient注解

> @EnableEurekaClient与@EnableDiscoveryClient,@EnableEurekaClient该组合注解包含了@EnableDiscoveryClient注解,但是在Finchley.RELEASE版本及之后两者不在是包含关系，spring cloud中discovery service有许多种实现（eureka、consul、zookeeper等等），@EnableDiscoveryClient基于spring-cloud-commons,@EnableEurekaClient基于spring-cloud-netflix，更简单的说就是如果选用的注册中心是eureka，那么推荐@EnableEurekaClient，如果是其他的注册中心，那么推荐使用@EnableDiscoveryClient。

```
@SpringBootApplication
@EnableEurekaClient
public class SpringcloudEurekaProviderOneApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringcloudEurekaProviderOneApplication.class, args);
    }
    
}
```

```
@SpringBootApplication
@EnableEurekaClient
public class SpringcloudEurekaProviderTwoApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringcloudEurekaProviderTwoApplication.class, args);
    }
    
}
```

5. 成功启动如下，为集群模式：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310145130.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310145142.png)

### 3.2.3 eureka-consumer配置

1. **springcloud-eureka-consumer**引入依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
    <version>1.4.7.RELEASE</version>
</dependency>
```

2. **springcloud-eureka-consumer**配置文件

```
eureka:
  client:
    fetchRegistry: true #注册到别的服务，所有设置为true
    registerWithEureka: true #需要去检索服务，所有设置为true
    service-url:
      defaultZone: http://localhost:9001/eureka,http://localhost:9002/eureka #服务注册中心的配置内容，指定服务注册中心的位置
  server:
    enable-self-preservation: false 
    eviction-interval-timer-in-ms: 60000

server:
  port: 7000
spring:
  application:
    name: eureka-consumer
```

3. 添加Controller访问层

```
@RestController
@RequestMapping("consumer")
public class ConsumerController {

    private final static String url = "http://eureka-provider/provider/get";

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("get")
    public Object get() {
        return restTemplate.getForObject(url, String.class);
    }

}
```

4. 注入RestTemplate

> 在你使用标注了@LoadBalanced的RestTemplate调用服务的时候，就是用了Ribbon，开启了负载均衡Ribbon默认使用的是RoundRobinRule轮询算法（每个服务按顺序请求一次）

* **Ribbon自带负载均衡策略：**

  - BestAvailableRule ：选择一个最小的并发请求的server
  - AvailabilityFilteringRule ：过滤掉那些因为一直连接失败的被标记为circuit tripped的后端server，并过滤掉那些高并发的的后端server（active connections 超过配置的阈值）
  - WeightedResponseTimeRule：根据相应时间分配一个weight，相应时间越长，weight越小，被选中的可能性越低
  - RetryRule ：对选定的负载均衡策略机上重试机制
  - RoundRobinRule：roundRobin方式轮询选择server
  - RandomRule：随机选择一个server
  - ZoneAvoidanceRule：复合判断server所在区域的性能和server的可用性选择server

> 当然，我们也可以指定使用Ribbon的某种负载均衡算法，或者也可以自定义负载均衡算法

<1>. **指定Ribbon的随机算法为负载均衡策略**

```
@Configuration
public class BeanConfig {

    /**
     * @Bean 将服务注册到Spring容器中
     * @LoadBalanced 实现负载均衡调用服务
     **/
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

    /**
     * 选择随机算法
     **/
    @Bean
    public RandomRule getRandomRule(){
        return new RandomRule();
    }
}

```

<2>. **自定义负载均衡算法**

```
public class MyRule extends AbstractLoadBalancerRule {

    private int total = 0;        //总共被调用的次数，目前要求每台被调用5次

    private int currentIndex = 0; //当前提供服务的机器号

    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            return null;
        }
        Server server = null;

        while (server == null) {
            if (Thread.interrupted()) {
                return null;
            }
            List<Server> upList = lb.getReachableServers();
            List<Server> allList = lb.getAllServers();

            int serverCount = allList.size();
            if (serverCount == 0) {
                return null;
            }

            if (total < 5) {
                server = upList.get(currentIndex);
                total++;
            } else {
                total = 0;
                currentIndex++;
                if (currentIndex >= upList.size()) {
                    currentIndex = 0;
                }
            }

            if (server == null) {
                /*
                 * The only time this should happen is if the server list were
                 * somehow trimmed. This is a transient condition. Retry after
                 * yielding.
                 */
                Thread.yield();
                continue;
            }

            if (server.isAlive()) {
                return (server);
            }

            // Shouldn't actually happen.. but must be transient or a bug.
            server = null;
            Thread.yield();
        }

        return server;

    }

    protected int chooseRandomInt(int serverCount) {
        return ThreadLocalRandom.current().nextInt(serverCount);
    }

    @Override
    public Server choose(Object key) {
        return choose(getLoadBalancer(), key);
    }

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {

    }
}
```

```
@Configuration
public class BeanConfig {

    /**
     * @Bean 将服务注册到Spring容器中
     * @LoadBalanced 实现负载均衡调用服务
     **/
    @Bean
    @LoadBalanced
    public RestTemplate getRestTemplate() {
        return new RestTemplate();
    }

    /**
     * 自定义负载均衡算法（调用五次）
     **/
    @Bean
    public MyRule getMyRule() {
        return new MyRule();
    }
}
```

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310155000.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310155017.png)

> 这个自定义的类不能放在@ComponentScan所扫描的当前包以及子包下，否则我们自定义的这个配置类就会被所有的Ribbon 客户端所共享，也就是我们达不到特殊化指定的目的了
>
> 可以使用@RibbonClient(name = "SPRINGCLOUD-EMPLOYEE-PROVIDER", configuration = MyRule.class)明确指定针对哪个服务进行负载均衡，而configuration指定负载均衡的算法具体实现类。当然也可以没有，即对所有服务生效。


5. 启动类添加@EnableEurekaClient注解

```
@SpringBootApplication
@EnableEurekaClient
public class SpringcloudEurekaConsumerApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringcloudEurekaConsumerApplication.class, args);
    }
}
```

6. 成功启动后，显示如下：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310145917.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310145932.png)

7. 通过RestTemplete远程调用eureka-provider的接口

   访问 [http://localhost:7000/consumer/get](http://localhost:7000/consumer/get) 显示如下：

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310150233.png)

![](https://gitee.com/FocusProgram/PicGo/raw/master/20200310150248.png)
  
# 4. Eureka和Zookeeper对比

> Eureka是基于AP原则构建，而ZooKeeper是基于CP原则构建；ZooKeeper基于CP，不保证高可用，如果zookeeper正在选举或者Zookeeper集群中半数以上机器不可用，那么将无法获得数据。Eureka基于AP，能保证高可用，即使所有机器都挂了，也能拿到本地缓存的数据。作为注册中心，其实配置是不经常变动的，只有发版和机器出故障时会变。对于不经常变动的配置来说，CP是不合适的，而AP在遇到问题时可以用牺牲一致性来保证可用性，既返回旧数据，缓存数据。