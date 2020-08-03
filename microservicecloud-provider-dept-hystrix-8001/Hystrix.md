### Hystrix介绍

- Hystrix是一个用于处理分布式系统延迟和容错的开源库。分布式系统中，依赖避免不了调用失败，比如超时，异常等。Hystrix能保证在出现问题的时候，不会导致整体服务失败，避免级联故障，以提高分布式系统的弹性。
- Hystrix类似一个“断路器”，当系统中异常发生时，断路器给调用返回一个符合预期的，可处理的FallBack，这样就可以避免长时间无响应或抛出异常，使故障不能再系统中蔓延，造成雪崩。

#### 服务熔断

- 熔断机制的注解是@HystrixCommand
- 熔断机制是应对雪崩效应的一种==链路保护机制==，一般存在于服务端
- 当扇出链路的某个服务出现故障或响应超时，会进行==服务降级==，进而==熔断该节点的服务调用==，快速返回“错误”的相应信息。、
- Hystrix的熔断存在阈值，缺省是5秒内20次调用失败就会触发

[Hystrix源码]: https://github.com/Netflix/Hystrix

##### 熔断案例

1. 构建一个新的provider module（如复制8001module）
2. pom.xml加入hystrix依赖（一定要配合Eureka）

```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix</artifactId>
        </dependency>
```

3. application.xml中配置端口和Eureka信息（必配）和其他框架的配置信息（可选，如mybatis）
4. 编写具体业务逻辑
5. controller类中，在需要配置Fallback的方法上加入@@HystrixCommand(fallbackMethod = "XXX")注解，XXX为FallBack方法名本例中作为测试所以抛出了异常

```java
    @ResponseBody
    @GetMapping("/dept/{id}")
    @HystrixCommand(fallbackMethod = "nullDeptFallBack")
    public Dept findById(@PathVariable("id")Integer id) {
        Dept dept = deptService.findById(id);
        if (null == dept){
            throw new RuntimeException("返回值为空！");
        }
        return dept;
    }
```

6. 根据需要配置FallBack的方法返回值编写代码

```java
 public Dept nullDeptFallBack(@PathVariable("id")Integer id) {
        System.out.println(111);
        return new Dept().setId(id).setDeptName("nullName").setDbSource("nullDB");
    }
```

7. 主启动类中加入@EnableCircuitBreaker注解
8. 开启服务，测试

#### 解耦与降级处理

##### 降级

- 当系统整体资源快不够的时候，忍痛将部分服务暂时关闭，带渡过难关后，再重新开启。
- 降级处理时在==客户端==完成的，与服务端没有关系
- 理解：所谓降级，一般是从==整体负荷==考虑，当某个服务熔断之后，服务器将不再被调用，此时客户端可以自己准备一个本地的FallBack回调，返回一个缺省值。这样做虽然服务水平下降，但好歹可用，比直接挂掉好。

##### 为什么要解耦

如果按照上面的熔断案例来做的话，Controller下的每个方法，都要给其编写一个FallBack方法，当方法慢慢变多，就会造成代码膨胀，一个是增加编写的工作量，另外一个也会增大维护的难度，代码的耦合度也会高，是十分不合理的，所以要将其解耦。

##### 解耦思路

因为服务端的是通过实现接口访问服务端的，如果在父接口上实现了FallBack方法，通过这样一种方式去维护起来就能实现解耦，也顺便完成了降级的机制。

##### 解耦&降级案例

1. 在api模块中新建实现了FallbackFactory<T>接口的类，其中泛型T就是我们需要维护其FallBack的接口方法，并实现其create方法，在create方法中返回实现了T的对象，使用匿名内部类实现T。==注意：这个类一定要加@Component注解！！这个类一定要加@Component注解！！这个类一定要加@Component注解！！==

```java
import com.XXX.entity.Dept;
import feign.hystrix.FallbackFactory;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
public class DeptClientServiceFallBackFactory implements FallbackFactory<DeptClientService> {
    public DeptClientService create(Throwable throwable) {
        return new DeptClientService() {
            public boolean addDept(Dept dept) {
                return false;
            }

            public List<Dept> findAll() {
                return null;
            }

            public Dept findById(Integer id) {
                return new Dept().setId(id).setDeptName("服务器跪了,").setDbSource("迟点来吧");
            }
        };
    }
}
```

2. 修改步骤1中传入的泛型T接口，添加@FeignClient(fallbackFactory = T.class)注解

```java
@FeignClient(value = "MICROSERVICECLOUD-DEPT",fallbackFactory = DeptClientServiceFallBackFactory.class)
public interface DeptClientService {

    @PostMapping("/dept")
    public boolean addDept(Dept dept);

    @GetMapping("/dept")
    public List<Dept> findAll();

    @GetMapping("/dept/{id}")
    public Dept findById(@PathVariable("id")Integer id);
}
```

3. 修改consumer feign模块的application.xml文件，开启hystrix（注：在IDEA中可能没有代码提示，开启的true也没有正常高亮，但好像不需要做额外操作也不影响结果）

```yml
feign:
  hystrix:
    enabled: true
```

4. 开启服务并测试

