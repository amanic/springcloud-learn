## Feign负载均衡

Feign是一个声明式WebService客户端，使用方法时定义一个接口并在上面添加注解即可。Feign支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装，使其支持SpringMVC和HttpMessageConverters。Feign可以与Eureka和Ribbon组合使用以支持负载均衡。

[Feign源码]: https://github.com/OpenFeign/Feign

### 使用案例

1. 新建Feign模块，加入依赖（其实跟80消费者差不多，主要是多了Feign依赖）

```xml
    <dependencies>
        <dependency>
            <groupId>com.XXX</groupId>
            <artifactId>microservice-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
        </dependency>
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
        </dependency>
        <!--热部署-->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>springloaded</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </dependency>
    </dependencies>
```

2. 因为Feign开发其实是面向接口编程，所以Feign接口可以放在api模块中供各模块使用，所以要在api模块中添加Feign依赖
3. 在api中编写接口，接口上添加@FeignClient注解，并通过value指定作用的微服务名

```java
@FeignClient(value = "MICROSERVICECLOUD-DEPT")
public interface DeptClientService {

    @PostMapping("/dept")
    public boolean addDept(Dept dept);

    @GetMapping("/dept")
    public List<Dept> findAll();

    @GetMapping("/dept/{id}")
    public Dept findById(@PathVariable("id")Integer id);
}

```

4. 在Feign模块中编写Controller，并注入FeignClient接口，直接调用service接口中的方法即可（因为声明Feign接口时已经指定过微服务，所以访问时会正确地找到微服务）

```java
@RestController
@RequestMapping("/consumer")
public class ConsumerDeptController {
    @Autowired
    private DeptClientService service;

    @PostMapping("/dept")
    public boolean addDept(Dept dept){
        return service.addDept(dept);
    }

    @GetMapping("/dept")
    public List<Dept> findAll(){
        return service.findAll();
    }

    @GetMapping("/dept/{id}")
    public Dept findById(@PathVariable("id")Integer id){
        return service.findById(id);
    }
}
```

5. 修改Feign模块的主启动类，加入@EnableFeignClients注解和@ComponentScan注解（主要是扫描api中声明的接口）

```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients(basePackages = {"com.XXX"})
@ComponentScan("com.XXX")
public class Consumer80Feign_APP {
    public static void main(String[] args) {
        SpringApplication.run(Consumer80Feign_APP.class,args);
    }
}
```

6. 启动后访问，即会按照轮询的方式调用provider集群

### 总结

- Feign通过接口方法调用REST服务，在Eureka中查找对应的服务
- Feign集成了Ribbon技术，所以也支持负载均衡（轮询）