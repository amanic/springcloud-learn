## Ribbon负载均衡

Spring Cloud Ribbon是基于Netflix Ribbon实现的一套==客户端==负载均衡工具。Ribbon会自动帮助你基于某种规则（简单轮询、随机连接等），也可以实现自定义的负载均衡算法。 

[Ribbon源码]: https://github.com/Netflix/Ribbon

### 负载均衡

- 英文名称：Load Balance，微服务或分布式集群中常用的一种应用

- 简单来说负载均衡就是将用户的请求ping平摊的分配到多个任务上，从而是系统达到HA（高可用）
- 两种负载均衡：
  1. 集中式LB：偏硬件，服务的消费方和提供方之间使用独立的LB设施，由该设施负责把访问请求以某种策略转发至服务的提供方。
  2. 进程内LB：偏软件， 将LB逻辑集成到消费方，消费方从服务注册中心指导哪些地址可用，再自己选择一个合适的服务器。

#### Ribbon初步配置

- Ribbon是客户端负载均衡工具！！！所以应该配置在客户端。

1. 加入依赖，因为Riboon需要依赖Eureka运行，所以要同时加入Eureka依赖

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-ribbon</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
```

2. 对实现类加入@LoadBalanced注解

```java
@Bean
@LoadBalanced
public RestTemplate getRestTemplate() {
        return  new RestTemplate();
    }
}
```

3. 在application.yml文件中配置向注册中心注册，如果是作为消费者模块不提供服务，不应该注册自己

```yml
eureka:
  client:
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002.com:7002/eureka/,http://eureka7003.com:7003/eureka/
    register-with-eureka: false				#作为消费者不提供服务，不应该注册自己
```

4. 主启动类中加入@EnableEurekaClient注解

```java
@SpringBootApplication
@EnableEurekaClient
public class Consumer80_APP {
    public static void main(String[] args) {
        SpringApplication.run(Consumer80_APP.class,args);
    }
}
```

5. 以上步骤1~4完成后即可在controller中直接通过服务名访问系统中的微服务，服务名作为URI

```java
private static final String URL_PREFIX = "http://MICROSERVICECLOUD-DEPT/";
```
#### Ribbon负载均衡实现

架构示意图：

![image-20200803144213797](/Users/martea/Library/Application Support/typora-user-images/image-20200803144213797.png)

##### 实现方法

目标：构建provider集群后consumer通过负载均衡轮询调用在Eureka中注册的服务

1. 构建集群，新开两个provider模块，将原provider的==代码部分和pom.xml中依赖照搬==到新的provider中
2. 将原provider中application.yml文件照搬到新provider，并修改端口号，若新的provider使用自己的数据库，则修改数据库信息（其他配置也一样，如修改别名）
3. 集群中服务名称必须一致！！！

```yml
spring:
  application:
    name: microservicecloud-dept   #同一集群下必须使用同一服务名！！！！！
```

4. 启动服务，进行测试

##### 总结

Ribbon其实就是一个软负载均衡的客户端组件，可以和其他需要请求的客户端结合使用。

### Ribbon核心组件IRule

IRule：根据特定算法从服务列表中选取一个需要访问的服务

#### 七大方法

==IRule是一个接口，七大方法是其自带的落地实现类==

- RoundRobinRule：轮询（默认方法）
- RandomRule：随机
- AvailabilityFilteringRule：先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，还有并发的连接数量超过阈值的服务，然后对剩余的服务进行轮询
- WeightedResponseTimeRule：根据平均响应时间计算服务的权重。统计信息不足时会按照轮询，统计信息足够会按照响应的时间选择服务
- RetryRule：正常时按照轮询选择服务，若过程中有服务出现故障，在轮询一定次数后依然故障，则会跳过故障的服务继续轮询。
- BestAvailableRule：先过滤掉由于多次访问故障而处于断路器跳闸状态的服务，然后选择一个并发量最小的服务
- ZoneAvoidanceRule：默认规则，符合判断server所在的区域的性能和server的可用性选择服务

#### 切换规则方法

只需在==配置类==中配置一个返回具体方法的bean即可

```java
@Bean
public IRule MyRule(){
        return new RandomRule();    
    }
```

### 自定义Ribbon负载均衡算法

#### 配置及包位置

1. 自定义的Ribbon算法类不能放在主启动类所在的包及子报下（确切来说是不能放在@ComponentScan注解的包及子包下），否则会被全局应用到Ribbon服务中。应该把自定义算法类放在另外新建的包下，且这个类应该是为==配置类==。（其实与普通切换负载均衡规则类似，只不过是位置不同而已，普通的可以放在主启动类所在的包，自定义的要放在外面的包下）
2. 主启动类添加@RibbonClient(name = "微服务名",configuration = XXX.class)注解指定需要用到负载均衡的微服务名及自定义算法的class对象。

```java
@SpringBootApplication
@EnableEurekaClient
@RibbonClient(name = "MICROSERVICECLOUD-DEPT",configuration = MyRule.class)
public class Consumer80_APP {
    public static void main(String[] args) {
        SpringApplication.run(Consumer80_APP.class,args);
    }
}
```

####通过修改源代码获得自定义算法

目标：每个服务调用5次后再进行轮询（调用次数不是很对，懒得改了)

```java
package com.Rules;

import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.ILoadBalancer;
import com.netflix.loadbalancer.RoundRobinRule;
import com.netflix.loadbalancer.Server;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Configuration;

import java.util.List;
import java.util.concurrent.atomic.AtomicInteger;


public class MyRule extends AbstractLoadBalancerRule {

    private AtomicInteger nextServerCyclicCounter;
    private static final boolean AVAILABLE_ONLY_SERVERS = true;
    private static final boolean ALL_SERVERS = false;
    private int total = 0;
    private int currentIndex = 0;

    private static Logger log = LoggerFactory.getLogger(RoundRobinRule.class);

    public MyRule() {
        nextServerCyclicCounter = new AtomicInteger(0);
    }

    public MyRule(ILoadBalancer lb) {
        this();
        setLoadBalancer(lb);
    }

    public Server choose(ILoadBalancer lb, Object key) {
        if (lb == null) {
            log.warn("no load balancer");
            return null;
        }

        Server server = null;
        int count = 0;
        while (server == null && count++ < 10) {
            List<Server> reachableServers = lb.getReachableServers();
            List<Server> allServers = lb.getAllServers();
            int upCount = reachableServers.size();
            int serverCount = allServers.size();

            if ((upCount == 0) || (serverCount == 0)) {
                log.warn("No up servers available from load balancer: " + lb);
                return null;
            }
            if (total > 5) {
                total = 0;
                int nextServerIndex = incrementAndGetModulo(serverCount);
                currentIndex = nextServerIndex;
                server = allServers.get(nextServerIndex);
            }else {
                if (currentIndex>=serverCount) {
                    currentIndex = 0;
                }
                server = allServers.get(currentIndex);
                total++;
            }


            if (server == null) {
                /* Transient. */
                Thread.yield();
                continue;
            }

            if (server.isAlive() && (server.isReadyToServe())) {
                return (server);
            }

            // Next.
            server = null;
        }

        if (count >= 10) {
            log.warn("No available alive servers after 10 tries from load balancer: "
                    + lb);
        }
        return server;
    }

    /**
     * Inspired by the implementation of {@link AtomicInteger#incrementAndGet()}.
     *
     * @param modulo The modulo to bound the value of the counter.
     * @return The next value.
     */
    private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextServerCyclicCounter.get();
            int next = (current + 1) % modulo;
            if (nextServerCyclicCounter.compareAndSet(current, next))
                return next;
        }
    }


    public Server choose(Object key) {
        return choose(getLoadBalancer(), key);
    }


    public void initWithNiwsConfig(IClientConfig clientConfig) {
    }
}

```



