# ring Cloud setinel

> 在微服务等分布式架构中，服务容错是老生常谈的问题了，我们都知道在微服务架构中会存在多个微服务，而绝大部分微服务之间都会存在调用关系。若由于某个底层服务不可用从而产生连锁反应，导致一系列的上层服务崩溃、故障，这种现象被称为雪崩效应或级联故障。所以在微服务等分布式架构中，能够防御服务雪崩效应的容错方案是必不可少的。

## 常见的容错方案

**1、超时限制：**

设置请求超时时间，让请求线程在等待超过一定的时间后就判定为请求失败从而释放请求线程，在某些场景下线程释放得够快的话，就不会因为不断创建线程而导致资源耗尽引起的服务崩溃。

**2、接口限流：**

例如，上图中的服务A只能承受1k左右的QPS，那么就设置一个最大请求数量阈值，当QPS达到1k时就拒绝在这之后的请求。就像是我只能吃一碗饭，就算给我三碗我也只吃一碗。

**3、舱壁模式：**

舱壁模式实际上就是借鉴于现实生活中的船舱结构而设计，一艘船想要不那么容易沉也需要具备有一定的”容错“能力，而早期的船由于设计上的欠缺，只要一个地方进水了，那么水就会逐渐漫进整个船舱，这种结构的船几乎没有“容错”能力，所以就比较容易沉。于是此时就有人想到将原本一体的船舱分隔成一个个独立的船舱，船舱之间都使用钢板焊死隔开，这些钢板就是所谓的舱壁了。采用这种设计后，就算当其中一个两个船舱进水了，也不会影响到其他船舱，这艘船依旧能够正常行驶。

在软件层面上借鉴这种思想，我们可以让每个服务都运行在自己独立的线程池中，线程池之间是互不干扰的，服务A的线程池资源耗尽也不会影响到服务B。此时线程池就像船舱的舱壁一样将不同的服务资源隔离开来，这样某个服务挂掉也不会影响其他服务的运行。

**4、断路器模式：**

断路器模式的思想实际上和家里的断路器一样，在软件层面大致就是对某个服务的API进行监控，若在一定时间内调用的失败率或失败次数达到指定的阈值就认为该API是不可用的从而触发“跳闸”，即此时断路器就处于打开状态。过了一段时间后断路器会处于一个半开状态，若在半开状态时尝试调用该API成功后就会关闭断路器，否则依旧认为不可用让断路器继续处于打开状态。



## Sentinel概述

[[官方文档](https://github.com/alibaba/Sentinel/wiki)]
Sentinel 是什么，官方描述如下：

随着微服务的流行，服务和服务之间的稳定性变得越来越重要。Sentinel 是面向分布式服务架构的轻量级流量控制、熔断降级框架，主要以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度来帮助您保护服务的稳定性。

Setinel特性是什么？

- **丰富的应用场景**：Sentinel 承接了阿里巴巴近 10 年的双十一大促流量的核心场景，例如秒杀（即突发流量控制在系统容量可以承受的范围）、消息削峰填谷、集群流量控制、实时熔断下游不可用应用等。
- **完备的实时监控**：Sentinel 同时提供实时的监控功能。您可以在控制台中看到接入应用的单台机器秒级数据，甚至 500 台以下规模的集群的汇总运行情况。
- **广泛的开源生态**：Sentinel 提供开箱即用的与其它开源框架/库的整合模块，例如与 Spring Cloud、Apache Dubbo、gRPC、Quarkus 的整合。您只需要引入相应的依赖并进行简单的配置即可快速地接入 Sentinel。同时 Sentinel 提供 Java/Go/C++ 等多语言的原生实现。
- **完善的 SPI 扩展机制**：Sentinel 提供简单易用、完善的 SPI 扩展接口。您可以通过实现扩展接口来快速地定制逻辑。例如定制规则管理、适配动态数据源等。

[![p90SPVP.png](https://s1.ax1x.com/2023/05/08/p90SPVP.png)](https://imgse.com/i/p90SPVP)



**Sentinel 分为两个部分:**

- **核心库（Java 客户端）**不依赖任何框架/库，能够运行于所有 Java 运行时环境，同时对 Dubbo / Spring Cloud 等框架也有较好的支持。
- **控制台（Dashboard）**基于 Spring Boot 开发，打包后可以直接运行，不需要额外的 Tomcat 等应用容器。

我们说的资源，可以是任何东西，服务，服务里的方法，甚至是一段代码。使用 Sentinel 来进行资源保护，主要分为几个步骤:

1. 定义资源
2. 定义规则
3. 检验规则是否生效

先把可能需要保护的资源定义好（埋点），之后再配置规则。也可以理解为，只要有了资源，我们就可以在任何时候灵活地定义各种流量控制规则。在编码的时候，只需要考虑这个代码是否需要保护，如果需要保护，就将之定义为一个资源。

对于主流的框架，我们提供适配，只需要按照适配中的说明配置，Sentinel 就会默认定义提供的服务，方法等为资源。



## 2.Setinel规则种类

Sentinel 的所有规则都可以在内存态中动态地查询及修改，修改之后立即生效。同时 Sentinel 也提供相关 API，供您来定制自己的规则策略。

Sentinel 支持以下几种规则：**流量控制规则**、**熔断降级规则**、**系统保护规则**、**来源访问控制规则** 和 **热点参数规则**。

### 2.1 流量控制规则(FlowRule)

- 同一个资源可以同时有多个限流规则，检查规则时会依次检查。
- 我们可以通过调用 FlowRuleManager.loadRules() 方法来用硬编码的方式定义流量控制规则，比如：

```java
/**
* 限流规则：
*
*/
private void initFlowRule() {
    List<FlowRule> rules = new ArrayList<>();
    FlowRule rule = new FlowRule();
    //资源名，资源名是限流规则的作用对象
    rule.setResource(RESOURCES_NAME);
    //限流阈值类型，QPS 模式（1）或并发线程数模式（0）
    rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
    // Set limit QPS to 20.
    rule.setCount(2);//限流阈值
    //流控针对的调用来源.默认default，代表不区分调用来源
    rule.setLimitApp("default");
    //调用关系限流策略：0:直接 1:关联 2:链路.默认根据资源本身（直接）
    rule.setStrategy(RuleConstant.STRATEGY_DIRECT);
    //流控效果，不支持按调用关系限流.0. default(reject directly), 1. warm up 2. rate limiter 3. warm up + rate limiter
    rule.setControlBehavior(0);
    //是否集群限流。默认false
    rule.setClusterMode(false);
    rules.add(rule);
    FlowRuleManager.loadRules(rules);
}
```

更多详细内容可以参考 [流量控制](https://github.com/alibaba/Sentinel/wiki/流量控制)。

### 2.2 熔断降级规则(DegradeRule)

- 同一个资源可以同时有多个降级规则。
- 通过调用 DegradeRuleManager.loadRules() 方法来用硬编码的方式定义流量控制规则。

```java
private void initDegradeRule() {
    List<DegradeRule> rules = new ArrayList<>();
    DegradeRule rule = new DegradeRule();
    rule.setResource(RESOURCES_NAME);//资源名，即规则的作用对象
    rule.setGrade(RuleConstant.DEGRADE_GRADE_RT);//熔断策略，支持慢调用比例/异常比例/异常数策略.默认慢调用比例
    // set threshold RT, 10 ms
    rule.setCount(10);//慢调用比例模式下为慢调用临界 RT（超出该值计为慢调用）；异常比例/异常数模式下为对应的阈值
    rule.setTimeWindow(10);//熔断时长，单位为 s
    rule.setMinRequestAmount(5);//熔断触发的最小请求数，请求数小于该值时即使异常比率超出阈值也不会熔断（1.7.0 引入）.默认5
    rule.setStatIntervalMs(1000);//统计时长（单位为 ms），如 60*1000 代表分钟级（1.8.0 引入）默认1000ms
    rule.setSlowRatioThreshold(0.5);//慢调用比例阈值，仅慢调用比例模式有效（1.8.0 引入）
    rules.add(rule);
    DegradeRuleManager.loadRules(rules);
}
```

更多详情可以参考 [熔断降级](https://github.com/alibaba/Sentinel/wiki/熔断降级)。

### 2.3 系统保护规则 (SystemRule)

Sentinel 系统自适应限流从整体维度对应用入口流量进行控制，结合应用的 Load、CPU 使用率、总体平均 RT、入口 QPS 和并发线程数等几个维度的监控指标，通过自适应的流控策略，让系统的入口流量和系统的负载达到一个平衡，让系统尽可能跑在最大吞吐量的同时保证系统整体的稳定性。

系统规则包含下面几个重要的属性：

```java
private void initSystemRule() {
    List<SystemRule> rules = new ArrayList<>();
    SystemRule rule = new SystemRule();
    rule.setHighestSystemLoad(10);//load1 触发值，用于触发自适应控制阶段.默认-1
    rule.setAvgRt(-1);//所有入口流量的平均响应时间.默认-1不生效
    rule.setMaxThread(-1);//入口流量的最大并发数.默认-1不生效
    rule.setQps(-1);//所有入口资源的 QPS.-1 (不生效)
    rule.setHighestCpuUsage(-1);//当前系统的 CPU 使用率（0.0-1.0） 默认-1不生效
    rules.add(rule);
    SystemRuleManager.loadRules(rules);
}
```

注意系统规则只针对入口资源（EntryType=IN）生效。更多详情可以参考 [系统自适应保护文档](https://github.com/alibaba/Sentinel/wiki/系统自适应限流)



### 2.4 来源访问控制规则

很多时候，我们需要根据调用方来限制资源是否通过，这时候可以使用 Sentinel 的访问控制（黑白名单）的功能。黑白名单根据资源的请求来源（origin）限制资源是否通过，若配置白名单则只有请求来源位于白名单内时才可通过；若配置黑名单则请求来源位于黑名单时不通过，其余的请求通过。

授权规则，即黑白名单规则（AuthorityRule）非常简单，主要有以下配置项：

- resource：资源名，即规则的作用对象
- limitApp：对应的黑名单/白名单，不同 origin 用 , 分隔，如 appA,appB
- strategy：限制模式，AUTHORITY_WHITE 为白名单模式，AUTHORITY_BLACK 为黑名单模式，默认为白名单模式

```java
private void initAuthorityRule(){
    List<AuthorityRule> rules = new ArrayList<>();
    AuthorityRule rule = new AuthorityRule();
    //资源名，即规则的作用对象
    rule.setResource("")；      
    //限制模式，AUTHORITY_WHITE 为白名单模式，AUTHORITY_BLACK 为黑名单模式，默认为白名单模式
    rule.setStrategy(RuleConstant.AUTHORITY_WHITE);   
    //对应的黑名单/白名单，不同 origin 用 , 分隔，如 appA,appB
    rule.setLimitApp("");
    rules.add(rule);
    AuthorityRuleManager.loadRules(rules);
}
```

更多详情可以参考 [来源访问控制](https://github.com/alibaba/Sentinel/wiki/黑白名单控制)。

### 2.5 热点参数规则

详情可以参考 [热点参数限流](https://github.com/alibaba/Sentinel/wiki/热点参数限流)。

------

## 3.Setinel使用

现在我们来为项目整合Sentinel，有两种方式

- 硬编码方式
- 注解方式

### 编码方式

```xml
<dependency>
  <groupId>com.alibaba.csp</groupId>
  <artifactId>sentinel-core</artifactId>
  <version>1.8.4</version>
</dependency>
```

```java
@RestController
public class SentinelController {
    
    @PostConstruct
    public void init() {
        List<FlowRule> rules = new ArrayList<>();
        FlowRule rule = new FlowRule();
        rule.setResource("HelloWorld");
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        // Set limit QPS to 20.
        rule.setCount(2);
        rules.add(rule);
        FlowRuleManager.loadRules(rules);
    }
    
    @GetMapping("/hello")
    public String hello(@RequestParam String message) { 
        Entry entry = null;
        try {
            entry = SphU.entry(message);
            /*您的业务逻辑 - 开始*/
            System.out.println("hello world");
            return "hello :" + message;
            /*您的业务逻辑 - 结束*/
        } catch (BlockException e1) {
            /*流控逻辑处理 - 开始*/
            System.out.println("block!");
            /*流控逻辑处理 - 结束*/
            throw new RuntimeException("接口被流控");
        } finally {
            if (entry != null) {
                entry.exit();
            }
        }
    }
}
```

### 注解方式

[[官方文档](https://github.com/alibaba/Sentinel/wiki/注解支持)]

Sentinel 提供了 @SentinelResource 注解用于定义资源，并提供了 AspectJ 的扩展用于自动定义资源、处理 BlockException 等。使用 [Sentinel Annotation AspectJ Extension](https://github.com/alibaba/Sentinel/tree/master/sentinel-extension/sentinel-annotation-aspectj) 的时候需要引入以下依赖

```xml
<dependency>
  <groupId>com.alibaba.csp</groupId>
  <artifactId>sentinel-annotation-aspectj</artifactId>
  <version>x.y.z</version>
</dependency>
```

上面都是针对硬编码的方式进行资源的定义，代码入侵严重，下面使用注解的方式进行定义资源。

Sentinel 支持通过 @SentinelResource 注解定义资源并配置 blockHandler 和 fallback 函数来进行限流之后的处理。示例

```java
// 原本的业务方法.
@SentinelResource(blockHandler = "blockHandlerForGetUser")
public User getUserById(String id) {
    throw new RuntimeException("getUserById command failed");
}

// blockHandler 函数，原方法调用被限流/降级/系统保护的时候调用
public User blockHandlerForGetUser(String id, BlockException ex) {
    return new User("admin");
}
```

**配置**

**Spring Cloud Alibaba**

若您是通过 [Spring Cloud Alibaba](https://github.com/alibaba/spring-cloud-alibaba/wiki/Sentinel) 接入的 Sentinel，则无需额外进行配置即可使用 @SentinelResource 注解。

**Spring AOP**

若您的应用使用了 Spring AOP（无论是 Spring Boot 还是传统 Spring 应用），您需要通过配置的方式将 SentinelResourceAspect 注册为一个 Spring Bean：

```java
@Configuration
public class SentinelAspectConfiguration {
    
    @Bean
    public SentinelResourceAspect sentinelResourceAspect() {
        return new SentinelResourceAspect();
    }
}
```



## Sentinel Dashboard控制台

> setinel提供了可视化的页面方便我们去进行观测机器的qps等运行情况，以及配置各种请求规则。这节就带领大家搭建一个setinel Dashboard控制台

Sentinel 控制台提供一个轻量级的[控制台](https://github.com/alibaba/Sentinel/wiki/控制台)，它提供机器发现、单机资源实时监控、集群资源汇总，以及规则管理的功能。您只需要对应用进行简单的配置，就可以使用这些功能。

Sentinel 控制台包含如下功能:

- [查看机器列表以及健康情况](https://github.com/alibaba/Sentinel/wiki/控制台#4-查看机器列表以及健康情况)：收集 Sentinel 客户端发送的心跳包，用于判断机器是否在线。
- [监控 (单机和集群聚合)](https://github.com/alibaba/Sentinel/wiki/控制台#5-监控)：通过 Sentinel 客户端暴露的监控 API，定期拉取并且聚合应用监控信息，最终可以实现秒级的实时监控。
- [规则管理和推送](https://github.com/alibaba/Sentinel/wiki/控制台#6-规则管理及推送)：统一管理推送规则。
- [鉴权](https://github.com/alibaba/Sentinel/wiki/控制台#鉴权)：生产环境中鉴权非常重要。这里每个开发者需要根据自己的实际情况进行定制。

**注意**：

1. Sentinel 控制台目前仅支持单机部署。Sentinel 控制台项目提供 Sentinel 功能全集示例，不作为开箱即用的生产环境控制台，不提供安全可靠保障。若希望在生产环境使用请根据[文档](https://github.com/alibaba/Sentinel/wiki/在生产环境中使用-Sentinel)自行进行定制和改造。
2. 集群资源汇总仅支持 500 台以下的应用集群，有大概 1 - 2 秒的延时。



### jar包方式搭建

> https://github.com/alibaba/Sentinel/releases

[![p90SiUf.png](https://s1.ax1x.com/2023/05/08/p90SiUf.png)](https://imgse.com/i/p90SiUf)

下载完成后，然后打开cmd，通过命令运行该jar包。如下：

```shell
java -jar sentinel-dashboard-1.6.3.jar
```

默认端口8080 可以指定启动端口

```shell
java -jar sentinel-dashboard-1.6.3.jar --server.port=8085
```

启动后页面如下.控制台用户密码 sentinel/sentinel

[![p90SAPS.png](https://s1.ax1x.com/2023/05/08/p90SAPS.png)](https://imgse.com/i/p90SAPS)

登录之后即可进入页面看到对应的服务信息。但是注意一点就是，这里面的配置默认是配置在内存中的，一旦setinel重启，所有的配置全部失效。这就需要了解setinel的持久化



## Setinel规则持久化

[[官方参考文档](https://github.com/alibaba/Sentinel/wiki/动态规则扩展)]

我们在使用setinel控制台的时候，如果关闭服务，所做的规则配置将全部丢失，这在生产环境中是无法容忍的，所以我们要考虑到将规则进行持久化。

Setinel的持久化分为三种模式：pull，push，以及原始模式。这里主要讲述push模式。生产环境建议使用push模式。

1. **原始模式**

- API 将规则推送至客户端并直接更新到内存中，扩展写数据源
- 简单，无任何依赖
- 不保证一致性；规则保存在内存中，重启即消失。严重不建议用于生产环境

1. **pull模式**

- 扩展写数据源， 客户端主动向某个规则管理中心定期轮询拉取规则，这个规则中心可以是 RDBMS、文件等。
- 简单，无任何依赖；规则持久化。
- 不保证一致性；实时性不保证，拉取过于频繁也可能会有性能问题。

1. **push模式**

- 扩展读数据源ReadableDataSource，规则中心统一推送，客户端通过注册监听器的方式时刻监听变化，比如使用 Nacos、Zookeeper 等配置中心。这种方式有更好的实时性和一致性保证。生产环境下一般采用 push 模式的数据源。
- 规则持久化；一致性；快速。
- 引入第三方依赖。

### Setinel push模式

生产环境建议使用push模式。通过控制台设置规则后将规则推送到统一的规则中心，客户端实现 ReadableDataSource 接口端监听规则中心实时获取变更，流程如下：

[![p90SE8g.png](https://s1.ax1x.com/2023/05/08/p90SE8g.png)](https://imgse.com/i/p90SE8g)

DataSource 扩展常见的实现方式有:

- **拉模式**：客户端主动向某个规则管理中心定期轮询拉取规则，这个规则中心可以是 RDBMS、文件，甚至是 VCS 等。这样做的方式是简单，缺点是无法及时获取变更；
- **推模式**：规则中心统一推送，客户端通过注册监听器的方式时刻监听变化，比如使用 [Nacos](https://github.com/alibaba/nacos)、Zookeeper 等配置中心。这种方式有更好的实时性和一致性保证。



### 如何实现push模式

> 下载Setinel源码修改Dashboard 规则。

Dashboard 规则改造主要通过2个接口：

```java
com.alibaba.csp.sentinel.dashboard.rule.DynamicRuleProvider
com.alibaba.csp.sentinel.dashboard.rule.DynamicRulePublisher
```

- Download [Sentinel Source Code](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Falibaba%2FSentinel)
- 修改原sentinel-dashboard项目下的POM文件

```xml
<!-- for Nacos rule publisher sample -->
<dependency>
  <groupId>com.alibaba.csp</groupId>
  <artifactId>sentinel-datasource-nacos</artifactId>
  <!--注释掉原文件中的scope，让其不仅在test的时候生效-->
  <!--<scope>test</scope>-->
</dependency>
```

1. 复制sentinel-dashboard项目下test下的nacos包src/test/java/com/alibaba/csp/sentinel/dashboard/rule/nacos到 src/main/java/com/alibaba/csp/sentinel/dashboard/rule下。
2. 修改NacosConfig中的指向Nacos 的ip和端口。

[![p90SmKs.png](https://s1.ax1x.com/2023/05/08/p90SmKs.png)](https://imgse.com/i/p90SmKs)


> **修改controller中的默认provider & publisher**

com.alibaba.csp.sentinel.dashboard.controller.v2.FlowControllerV2中

```java
@RestController
@RequestMapping(value = "/v2/flow")
public class FlowControllerV2 {
    
    @Autowired
    //@Qualifier("flowRuleDefaultProvider")
    @Qualifier("flowRuleNacosProvider")
    private DynamicRuleProvider<List<FlowRuleEntity>> ruleProvider;
    
    @Autowired
    //@Qualifier("flowRuleDefaultPublisher")
    @Qualifier("flowRuleNacosPublisher")
    private DynamicRulePublisher<List<FlowRuleEntity>> rulePublisher;
    
}
```



[![p90SZvj.png](https://s1.ax1x.com/2023/05/08/p90SZvj.png)](https://imgse.com/i/p90SZvj)



>  打开 /Sentinel-1.6.2/sentinel-dashboard/src/main/webapp/resources/app/scripts/directives/sidebar/sidebar.html文件

```html
<!--<li ui-sref-active="active" ng-if="entry.appType==0">-->
    <!--<a ui-sref="dashboard.flow({app: entry.app})">-->
        <!--<i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则 V1</a>-->
<!--</li>-->

------------改为

<li ui-sref-active="active" ng-if="entry.appType==0">
    <a ui-sref="dashboard.flow({app: entry.app})">
        <i class="glyphicon glyphicon-filter"></i>&nbsp;&nbsp;流控规则 Nacos</a>
</li>
```

[![p90Sl5T.png](https://s1.ax1x.com/2023/05/08/p90Sl5T.png)](https://imgse.com/i/p90Sl5T)

Dashboard中要修改的代码已经好了。

重新启动

```shell
Sentinel-dashboard mvn clean package -DskipTests
```

### 客户端使用

```xml
<!-- actuator，用于暴露监控端点 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.csp</groupId>
    <artifactId>sentinel-datasource-nacos</artifactId>
</dependency>
```



bootstrap.yml

```yaml
spring:
  application:
    name: springcloud-order
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
      discovery:
        server-addr: 127.0.0.1:8848
    sentinel:
      transport:
        dashboard: 127.0.0.1:8080
      datasource:
        yibo_flow:
          nacos:
            server-addr: 127.0.0.1:8848
            dataId: ${spring.application.name}-flow-rules
            groupId: SENTINEL_GROUP
            rule_type: flow
```

application.yml

```yaml
server:
  port: 7003
logging:
  level:
    com.itmck.feign: debug

# 暴露所有端点
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
```

访问nacos并创建

[![p90SNrR.png](https://s1.ax1x.com/2023/05/08/p90SNrR.png)](https://imgse.com/i/p90SNrR)

发布之后在sentinel控制台可以看到规则

[![p90SdVx.png](https://s1.ax1x.com/2023/05/08/p90SdVx.png)](https://imgse.com/i/p90SdVx)

通过sentinel控制台修改流控规则在nacos配置中可以看到实时修改

## 微服务下setinel使用场景

> 随着微服务的流行，服务和服务之间的稳定性变得越来越重要。 [Sentinel](https://github.com/alibaba/Sentinel) 以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度保护服务的稳定性。

**springcloud alibaba集成setinel使用如下依赖即可**

```xml
<!--添加sentinel依赖-->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

如果与openfeign一起使用，想达到熔断效果，可以在yaml配置如下

```yaml
feign:
  sentinel:
    enabled: true
```

具体代码参考 `Spring Cloud Openfeign`章节

### 全部依赖说明

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>springcloud-parent</artifactId>
        <groupId>con.itmck</groupId>
        <version>1.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>springcloud-sentinel</artifactId>

    <dependencies>
        <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
        </dependency>
        <!-- actuator，用于暴露监控端点 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
            <groupId>com.alibaba.csp</groupId>
            <artifactId>sentinel-datasource-nacos</artifactId>
        </dependency>
    </dependencies>
</project>
```



### 暴露所有端点

```yaml
management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      show-details: always
```

所以接着到项目中整合一下Sentinel Dashboard的请求地址，在配置文件中添加如下配置：

```yaml
spring:
  cloud:
    sentinel:
      transport:
        # 配置sentinel控制台的地址
        dashboard: 127.0.0.1:8080
```

### 流控规则演示

- - 直接：当前资源的QPS达到设定的阈值，就触发限流
- 关联：当关联的资源的QPS达到设定的阈值，就触发限流。例如，/shares/1关联了/query，那么/query达到阈值，就会对/shares/1限流
- 链路：只记录指定链路上的流量，这种模式是针对接口级别的来源进行限流
  链路模式稍微有些抽象，这里举个简单的例子说明一下。下图中有两个调用链路，图中的/test-b和/test-a实际就是两个接口，它们都调用了同一个common资源，所以/test-b和/test-a就称为common的入口资源

```java
@Slf4j
@Api(value = "ConsumerController", tags = "控制器作用")
@RestController
public class ConsumerController {
 
    @Resource
    private MicroFeginClient microFeginClient;
    /**
     * fallback：失败调用，若本接口出现未知异常，则调用fallback指定的接口。
     * blockHandler：sentinel定义的失败调用或限制调用，若本次访问被限流或服务降级，则调用blockHandler指定的接口。
     */
    @SentinelResource(
            value = "hello2",
            blockHandler = "blockHandlerFunc",
            fallback = "fallbackFunc",
            blockHandlerClass = SetinelHandler.class
    )
    @ApiOperation(value = "hello", notes = "演示1")
    @GetMapping("/hello")
    public ApiResultResponse fec() {
        String hello = microFeginClient.hello();
        return ApiResultResponse
                .builder()
                .result(hello)
                .success(true)
                .build();
    }
}
```

- 处理限流降级方法 必须为static

```java
@Slf4j
public class SetinelHandler {
    public static ApiResultResponse blockHandlerFunc(String a, BlockException e) {
        // 如果被保护的资源被限流或者降级了，就会抛出BlockException
        log.warn("资源被限流.", e);
        return ApiResultResponse
                .builder()
                .message("接口限流了")
                .errorCode(100)
                .build();
    }
    public static ApiResultResponse fallbackFunc(String a) {
 
        log.warn("资源被降级了.");
        return ApiResultResponse
                .builder()
                .message("资源被降级了")
                .errorCode(100)
                .build();
    }
}
```

- 全局处理误信息进行相应优化,将上面替换成下面
  Sentinel默认在当前服务触发限流或降级时仅返回简单的异常信息,所以我们需要对错误信息进行相应优化，以便可以细致区分触发的是什么规则

```java
@Slf4j
@RestController
public class OrderSentinelController {


    //这里不用使用 @SentinelResource,也能被识别,会走自定义sentinel全局异常处理器
    @GetMapping("/order/sentinel/getOrder")
    public ApiResultResponse<String> getOrder(@RequestParam String name) {
        log.info("模拟sentinel进行接口流控,入参:{}", name);
        return ApiResultResponse.ok(name);
    }

}

```

实现BlockExceptionHandler重写handle

```java
@Slf4j
@Component
public class SentinelBlockHandler implements BlockExceptionHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, BlockException e) throws Exception {

        log.info("BlockExceptionHandler BlockException================" + e.getRule());
        ApiResultResponse<String> resp;
        // 不同的异常返回不同的提示语
        if (e instanceof FlowException) {
            resp = ApiResultResponse.error("接口限流", "4001");
        } else if (e instanceof DegradeException) {
            resp = ApiResultResponse.error("服务降级", "4002");
        } else if (e instanceof ParamFlowException) {
            resp = ApiResultResponse.error("热点参数限流", "4003");
        } else if (e instanceof SystemBlockException) {
            resp = ApiResultResponse.error("触发系统保护规则", "4004");
        } else if (e instanceof AuthorityException) {
            resp = ApiResultResponse.error("授权规则不通过", "4005");
        } else {
            resp = ApiResultResponse.error("系统异常", "500");
        }
        response.setStatus(500);
        response.setCharacterEncoding("utf-8");
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        new ObjectMapper().writeValue(response.getWriter(), resp);
    }
}
```

```java
@Data
@Builder
public class ApiResultResponse<T> implements Serializable {

    /**
     * 错误信息
     */
    private String message;

    /**
     * 错误码
     */
    public String errorCode;

    /**
     * 是否成功
     */
    private boolean success;

    /**
     * 响应体
     */
    private T result;

    public ApiResultResponse() {
    }

    public ApiResultResponse(String message, String errorCode, boolean success, T result) {
        this.message = message;
        this.errorCode = errorCode;
        this.success = success;
        this.result = result;
    }

    public static <T> ApiResultResponse<T> ok(T data) {
        return new ApiResultResponse<>("ok", "200", true, data);
    }

    public static <T> ApiResultResponse<T> ok(String message, T data) {
        return new ApiResultResponse<>(message, "200", true, data);
    }

    public static <T> ApiResultResponse<T> error(String message, String errorCode) {
        return new ApiResultResponse<>(message, errorCode, false, null);
    }
}
```

