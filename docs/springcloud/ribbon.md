# Spring Cloud Ribbon

> Spring Cloud Ribbon是一个基于HTTP和TCP的客户端负载均衡工具,它提供了许多负载均衡算法，如轮询，随机选择等.Ribbon通过使用客户端库来避免在客户端和服务器之间进行数据中转，从而最大程度地提高了性能和并发性能。简单来说他是一款客户端负载均衡工具。

## ribbon负载均衡策略

**Ribbon框架自带的负载策略类**

- com.netflix.loadbalancer.RoundRobinRule  - 轮询

- com.netflix.loadbalancer.RandomRule  - 随机

- com.netflix.loadbalancer.RetryRule - 重试，先按RoundRobinRule进行轮询，如果失败就在指定时间内进行重试

- com.netflix.loadbalancer.WeightedResponseTimeRule - 权重，响应速度越快，权重越大，越容易被选中。

- com.netflix.loadbalancer.BestAvailableRule  - 先过滤掉不可用的处于断路器跳闸转态的服务，然后选择一个并发量最小的服务

- com.netflix.loadbalancer.AvailabilityFilteringRule - 先过滤掉故障实例，再选择并发量较小的实例

- com.netflix.loadbalancer.ZoneAvoidanceRule - 默认规则，复合判断server所在区域的性能和server的可用性进行服务的选择。

## 使用案例

> 现有服务A订单服务，服务B为库存服务。调用链路是 A调用B

订单服务A，需要使用**RestTemplate**进行服务的调用.其中 RestTemplate是Spring提供。

注意:

> spring-cloud-starter-alibaba-nacos-discovery默认引入了ribbon的依赖。无需再单独引入

1. 使用前需要初始化@Bean交给Spring容器进行管理.
2. 使用@LoadBalanced 开启客户端负载均衡

```java
@Configuration
public class RestTemplateConfig {

    @LoadBalanced//开启负载均衡
    @Bean
    RestTemplate getRestTemplate() {
        return new RestTemplate();
    }
}
```

```java
@RequestMapping("/productA")
@RestController
public class RibbonController {


  	//注入RestTemplate
    @Resources
    private RestTemplate restTemplate;

    @GetMapping("/hello")
    public String hello(@RequestParam String name) {
        return restTemplate.getForObject("http://my-product-B/productB/hello?name={name}", String.class, name);
    }
}
```

服务B为库存服务。无需额外配置,仅仅只要启动服务,将服务注册到nacos注册中心.下面写一个简单rest接口

```java
@RestController
@Slf4j
@RequestMapping("/productB")
public class RibbonController {


    @Value("${server.port}")
    private String port;

    @GetMapping("/hello")
    public String hello(@RequestParam String name) {
        return "hello " + name + " come from :" + port;
    }
}
```

**测试负载均衡**

1. 库存服务B，分别使用端口8001,8002启动2个实例

2. 启动订单服务A,设置端口为7001,并调用 http://localhost:7001/productA/hello?name=miaock

3. 多次调用打印如下

   ```
   hello mck come from : 8001
   hello mck come from : 8002
   ```

   