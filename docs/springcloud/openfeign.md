# Spring Cloud Openfeign

> openfeign 声明式http客户端.使得我们调用远程API变得简单,只需简单的创建一个interface标注注解就能完成远程调用.默认整合ribbon实现负载均衡

- OpenFeign的设计宗旨式简化Java Http客户端的开发。Feign在restTemplate的基础上做了进一步的封装，由其来帮助我们定义和实现依赖服务接口的定义。在OpenFeign的协助下，我们只需创建一个接口并使用注解的方式进行配置（类似于Dao接口上面的Mapper注解）即可完成对服务提供方的接口绑定，大大简化了Spring cloud Ribbon的开发，自动封装服务调用客户端的开发量。
- OpenFeign集成了Ribbon,利用ribbon维护了服务列表，并且通过ribbon实现了客户端的负载均衡。与ribbon不同的是，通过OpenFeign只需要定义服务绑定接口且以申明式的方法，优雅而简单的实现了服务调用.

![image-20230506170006294.png](https://s2.loli.net/2023/05/08/PYINCZHxLMRgzsr.png)



## 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

## **创建Fegin客户端**

```java
//调用服务service-tow，当服务不可用调用ServiceFeignClientImpl其中。ServiceFeignClientImpl实现了ServiceFeignClient
@Component
@FeignClient(name = "service-tow", fallback = ServiceFeignClientImpl.class)
public interface ServiceFeignClient {

    @GetMapping(value = "/hello")
    String hello(@RequestParam("name") String name);

    @GetMapping(value = "/hello2")
    @ResponseBody
    Student hello2(@RequestParam("name") String name);

}

```

- @FeignClient(value = "在注册中心的服务名称",fallback="服务不可用调用")
- 如果你在项目里面设置了统一的请求路径(server.servlet.context-path),需要将@FeignClient注解调整@FeignClient(value = "服务名",path = "xxx")
- Feign 里面定义的接口，有多个@RequestParam，但只能有不超过一个@RequestBody

```java
@Component
public class ServiceFeignClientImpl implements ServiceFeignClient{

    @Override
    public String hello(@RequestParam String name) {
        return "被调用的服务挂了";
    }

    @Override
    public Student hello2(String name) {
        return null;
    }
}

```

配置如下：

```yaml
spring:
  application:
    name: productA #当前应用名
  cloud:
    nacos:
      discovery:
        server-addr: 101.43.70.178:8848 #注册中心地址,集群以逗号隔开
      config:
        server-addr: 101.43.70.178:8848 #配置中心地址,集群以逗号隔开
        file-extension: yaml
server:
  port: 7001
feign:
  hystrix:
    enabled: true  # 当依赖的服务不可用的时候走，feign默认集成了hystrix 所以配置这个
```

现关闭service-tow 请求：localhost:7001/feign/hello?name=mck

返回结果：

```
被调用的服务挂了
```

如果不配置如下配置

```yaml
feign:
  hystrix:
    enabled: true  # 当依赖的服务不可用的时候走
```

当服务挂了会返回如下提示：

```
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Sun May 07 14:33:23 CST 2023
There was an unexpected error (type=Internal Server Error, status=500).
com.netflix.client.ClientException: Load balancer does not have available server for client: service-tow
```

如果当前服务集成了setinel

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

则配置可以改为下面，可以达到同等效果

```yaml
feign:
  sentinel:
    enabled: true
```

## @EnableFeignClients

```java
@EnableDiscoveryClient
@SpringBootApplication
@EnableFeignClients("com.itmck.microicrconsumer.fegin")
public class MicroIcrConsumerApplication {
 
    public static void main(String[] args) {
        SpringApplication.run(MicroIcrConsumerApplication.class, args);
    }
 
}
```

注入并使用

```java
@RestController
public class FeignController {


    @Resource
    ServiceFeignClient serviceFeignClient;


    @GetMapping("/feign/hello")
    public String hello(@RequestParam String name) {
        return serviceFeignClient.hello(name);
    }

}
```



创建服务service-tow

配置如下：

```yaml
server:
  port: 7002
spring:
  application:
    name: service-tow  #当前应用名
  cloud:
    nacos:
      discovery:
        server-addr: 101.43.70.178:8848 #注册中心地址,集群以逗号隔开
      config:
        server-addr: 101.43.70.178:8848 #配置中心地址,集群以逗号隔开
        file-extension: yaml

```

代码如下，提供一个接口

```java
@SpringBootApplication
@EnableDiscoveryClient
@RestController
public class SpringCloudServiceTowApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringCloudServiceTowApplication.class, args);
    }



    @Value("${server.port}")
    private String port;

    @GetMapping("/hello")
    public String hello(@RequestParam String name) {
        return "hello " + name + " come from :" + port;
    }

}

```







**总结:**

在我们是一个feigin的时候是要保证服务都是在一个注册中心。fegin就是对http的封装方便我们去使用。

其实还有个开源的http封装的框架使用也很灵活,可以参考http://forest.dtflyx.com/docs/

## gn高级使用

> 我们使用Fegin的时候并不是简单的集成,还有一些特殊的需求,如,日志打印,Fegin接口密码认证等

### Feign 自定义日志

我们在开发时候一般出现问题，需要将出错信息、异常信息以及正常的输入输出打印出来，以供我们更好的排查和解决问题，比如想看到接口的性能，就需要看Feign 的日志，那么如何让Feign 的日志展示出来呢

```java
@Configuration
public class FeignConfig {
 
    /**
     * 输出的日志级别
     * @return
     */
    @Bean
    Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}
```

新建FeignConfig类，并设置日志级别的输出信息

其中:日志等级:

- NONE： 不输出日志
- BASIC： 只输出请求方法的URL 和响应状态码以及接口的请求时间
- HEADERS ：将 BASIC信息和请求头信息输出
- FULL ：输出完 的请求信息

```java
//将配置信息添加到Feign 的接口配置上面 configuration = FeignConfig.class
@FeignClient(name = "micro-icr-risk",configuration = FeignConfig.class)
public interface MicroFeginClient {

}
```

**application.yml中设置日志级别**

```yaml
logging:
  level:
    com.yisu: debug  #com.yisu是设置包路径，在这个路径里面的debug信息都会被捕获到
```

### Fegin的Basic认证

> 接口上都会设置需要的权限信息，而一般的权限认证有通过token校验的、也有通过用户名密码校验的等方式。比如我们在Feign 请求中我们可以配置Basic 认证，如下

```java
/**
 * 设置Spring Security Basic认证的用户名密码
 * @return
 */
@Bean
public BasicAuthRequestInterceptor basicAuthRequestInterceptor(){
    return new BasicAuthRequestInterceptor("user","123456");
}
```

### Feign 自定义认证配置

```java
@NoArgsConstructor
public class FeignAuthRequestInterceptor implements RequestInterceptor {
 
    @Override
    public void apply(RequestTemplate requestTemplate) {
        //编写自己的业务逻辑
    }
}
```

然后在FeignConfig配置中添加Bean

```java
@Bean
public FeignAuthRequestInterceptor basicAuthRequestInterceptor(){
    return new FeignAuthRequestInterceptor();
}

```

