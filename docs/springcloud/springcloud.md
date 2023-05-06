

# 什么是微服务

**微服务架构是一种架构概念**，指将一个大型应用程序拆分成多个小型服务，并通过互联网协议进行通信。每个服务都独立部署、运行、迭代，由一个小团队负责开发维护，可以独立扩展和升级，从而实现更高效、更灵活的开发和部署。微服务架构可以使得应用系统更具弹性，更容易适应不同的需求变化和规模变化。



微服务架构优点：

1. 高可靠性：微服务架构可以将应用拆分成多个小的模块，这样当某个模块发生故障时不会影响整个应用的正常运行。

2. 独立性：每个微服务都是独立的，可以单独进行开发、测试、部署和扩展，提高开发团队的灵活性。

3. 易于维护和升级：微服务架构使得应用的组件化部署和更新变得非常容易，不需要对整个应用进行重大改动。

4. 高扩展性：微服务架构能够根据业务需求灵活扩展，增加或减少服务组件。

5. 技术异构性：微服务架构允许不同的服务使用不同的语言、框架和技术栈进行开发，能够充分利用各种技术的优势，提高开发效率和质量。

微服务架构缺点：

1. 增加系统复杂度：微服务架构涉及到多个服务的协作，可能导致系统的复杂度增加，需要增加管理和监控的工作量。

2. 服务间通信：微服务架构中的服务需要通过网络进行通信，这可能导致性能瓶颈和网络延迟问题。

3. 分布式事务：由于微服务架构的服务是独立的，需要处理分布式事务问题，这增加了开发和维护的难度。

4. 测试难度：由于微服务架构涉及多个服务的协作，测试成为了一个更加困难的问题。

5. 运维工作量：微服务架构中的复杂性需要更多的人力资源来进行运维和监控，这增加了运维的工作量。



# CAP理论

**CAP 理论**

> 一个分布式系统最多只能同时满足一致性,可用性和分区容错性这三项中的两项。
>
> 备注：一致性(Consistency),可用性(Availability)和分区容错性(Partition tolerance)

- 一致性：所有节点在同一时间的数据完全一致。

- 可用性：服务一直可用，而且是正常响应时间

- 分区容错性：即分布式系统在遇到某节点或网络分区故障的时候，仍然能够对外提供满足一致性和可用性的服务

**CAP 权衡**

> 通过 CAP 理论，我们知道无法同时满足一致性、可用性和分区容错性这三个特性，那要舍弃哪个呢？

- 对于多数大型互联网应用的场景，主机众多、部署分散，而且现在的集群规模越来越大，所以节点故障、网络故障是常态， 而且要保证服务可用性达到 N 个 9，即保证 P 和 A，舍弃C（退而求其次保证最终一致性）。 虽然某些地方会影响客户体验，但没达到造成用户流程的严重程度。
- 对于涉及到钱财这样不能有一丝让步的场景，C 必须保证。网络发生故障宁可停止服务，这是保证 CA，舍弃 P。 貌似这几年国内银行业发生了不下 10 起事故，但影响面不大，报到也不多，广大群众知道的少。
- 还有一种是保证 CP，舍弃 A。例如网络故障是只读不写。 孰优孰略，没有定论，只能根据场景定夺，适合的才是最好的。



# BASE理论

> eBay 的架构师 Dan Pritchett 源于对大规模分布式系统的实践总结，在 ACM 上发表文章提出 BASE 理论，BASE 理论是对 CAP 理论的延伸，核心思想是即使无法做到强一致性（Strong Consistency，CAP 的一致性就是强一致性），但应用可以采用适合的方式达到最终一致性（Eventual Consitency）。

**基本可用（Basically Available）**

基本可用是指分布式系统在出现故障的时候，允许损失部分可用性，即保证核心可用。

电商大促时，为了应对访问量激增，部分用户可能会被引导到降级页面，服务层也可能只提供降级服务。这就是损失部分可用性的体现。

**软状态（Soft State）**

软状态是指允许系统存在中间状态，而该中间状态不会影响系统整体可用性。分布式存储中一般一份数据至少会有三个副本，允许不同节点间副本同步的延时就是软状态的体现。mysql replication 的异步复制也是一种体现。

**最终一致性（Eventual Consistency）**

最终一致性是指系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。弱一致性和强一致性相反，最终一致性是弱一致性的一种特殊情况。

**ACID 和 BASE 的区别与联系**

1. ACID 是传统数据库常用的设计理念，追求强一致性模型。
2. BASE 支持的是大型分布式系统，提出通过牺牲强一致性获得高可用性。
3. ACID 和 BASE 代表了两种截然相反的设计哲学，在分布式系统设计的场景中，系统组件对一致性要求是不同的，因此 ACID 和 BASE 又会结合使用。



# 微服务带来的挑战

> 一项技术的出现必定伴随着一定的问题。微服务也不例外。虽然说解决了耦合性，提高了独立性，高可用性，高扩展性，但是也带来了分布式治理，分布式通讯，分布式事务，分布式缓存等等一系列问题。

最常见的几大类问题：

- 服务这么多如何治理
- 服务之间如何通讯
- 服务挂了怎么办
- 服务之间事务如何解决（分布式事务）

**Spring Cloud Netflix**提供了一套微服务的解决方案，但是随着其进入维护阶段，**Spring Cloud Alibaba**接着提供了一套解决方案。

常见的鹅厂也推出了自己的解决方案 Spring Cloud Tencent。归根结底还是为了解决上述几大类问题。

# Spring Cloud

提起微服务，不得不提 Spring Cloud 全家桶系列，SpringCloud 是若干个框架的集合，包括 spring-cloud-config、spring-cloud-bus 等近 20 个子项目，提供了服务治理、服务网关、智能路由、负载均衡、断路器、监控跟踪、分布式消息队列、配置管理等领域的解决方案。

Spring Cloud 通过 Spring Boot 风格的封装，屏蔽掉了复杂的配置和实现原理，最终给开发者留出了一套简单易懂、容易部署的分布式系统开发工具包。

一般来说，Spring Cloud 包含以下组件，主要以 Netflix 开源为主：

![img](https://pic4.zhimg.com/v2-2d78afcc4e7019b9788478345059a9a3_r.jpg)





# Spring Cloud Alibaba

> 官方地址：https://spring.io/projects/spring-cloud-alibaba
>
> Spring Cloud Alibaba 是一套微服务解决方案，包含开发分布式应用微服务的必需组件，方便开发者通过 Spring Cloud 编程模型轻松使用这些组件来开发分布式应用服务。
>
> 依托 Spring Cloud Alibaba，您只需要添加一些注解和少量配置，就可以将 Spring Cloud 应用接入阿里微服务解决方案，通过阿里中间件来迅速搭建分布式应用系统。

核心功能：

- 服务治理
- 服务限流降级
- 服务消息驱动
- 分布式事务
- 阿里云对象存储
- 等等

# Nacos

> nacos官方文档地址: https://nacos.io/zh-cn/docs/quick-start.html
>
> Nacos /nɑ:kəʊs/ 是 Dynamic Naming and Configuration Service的首字母简称，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。
>
> Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。
>
> Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以“服务”为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。
>
> 一句话描述:集 注册中心+配置中心+服务管理 的平台.管理所有微服务、解决微服务之间调用关系错综复杂、难以维护的问题；

## Nacos核心功能点

**服务注册和发现**

Nacos Client会通过发送REST请求的方式向Nacos Server注册自己的服务，提供自身的元数据，比如ip地址、端口等信息。Nacos Server接收到注册请求后，就会把这些元数据信息存储在一个双层的内存Map中。

在服务注册后，Nacos Client会维护一个定时心跳来持续通知Nacos Server，说明服务一直处于可用状态，防止被剔除。默认5s发送一次心跳。

服务消费者（Nacos Client）在调用服务提供者的服务时，会发送一个REST请求给Nacos Server，获取上面注册的服务清单，并且缓存在Nacos Client本地，同时会在Nacos Client本地开启一个定时任务定时拉取服务端最新的注册表信息更新到本地缓存。

**服务健康检查**

Nacos Server会开启一个定时任务用来检查注册服务实例的健康情况，对于超过15s没有收到客户端心跳的实例会将它的healthy属性置为false(客户端服务发现时不会发现)，如果某个实例超过30秒没有收到心跳，直接剔除该实例(被剔除的实例如果恢复发送心跳则会重新注册)。

**服务同步**

Nacos Server集群之间会互相同步服务实例，用来保证服务信息的一致性。

**服务及其元数据管理**

Nacos 能让您从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA 以及最首要的 metrics 统计数据。

**动态配置服务**

动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。

## 安装nacos控制台

![image-20230505160247182.png](https://s2.loli.net/2023/05/06/JUj54TwQ7c93oKa.png)

**源码下载方式**

```shell
git clone https://github.com/alibaba/nacos.git
cd nacos/
mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U  
ls -al distribution/target/

// change the $version to your actual path
cd distribution/target/nacos-server-$version/nacos/bin
```

**安装包下载方式**

您可以从 [最新稳定版本](https://github.com/alibaba/nacos/releases) 下载 `nacos-server-$version.zip` 包。

```shell
  unzip nacos-server-$version.zip 或者 tar -xvf nacos-server-$version.tar.gz
  cd nacos/bin
```

![image-20230505160544835.png](https://s2.loli.net/2023/05/06/mrJcVfBvy97R2gp.png)

**启动服务**

> 注意直接启动是基于内存的形式保存配置等信息，生产环境需要修改，这里暂时体验

### Linux/Unix/Mac

启动命令(standalone代表着单机模式运行，非集群模式):

```shell
sh startup.sh -m standalone
```

如果您使用的是ubuntu系统，或者运行脚本报错提示[[符号找不到，可尝试如下运行：

```shell
bash startup.sh -m standalone
```

### Windows

启动命令(standalone代表着单机模式运行，非集群模式):

```shell
startup.cmd -m standalone
```

**访问**

```shell
访问 localhost:8848/nacos   
用户名/密码 nacos/nacos
```

![image-20230505162014203.png](https://s2.loli.net/2023/05/06/TbUSIpH4G3rsfAd.png)

## Nacos注册中心架构

![image-20230506102831827.png](https://s2.loli.net/2023/05/06/6Q4NBsd1bctuMKo.png)



## 版本对应关系

> 避坑指南：springcloud版本更新太快，学习springcloud首先就是整清楚版本依赖关系。避免出现莫名其妙的错误。

1. 说明:springcloudalibaba依赖与spring-cloud
2. 版本对应关系说明:https://github.com/alibaba/spring-cloud-alibaba/wiki/版本说明

![image-20230505163735187.png](https://s2.loli.net/2023/05/06/Xrw4aWetJv915PQ.png)

类似上面标记，每行代表的就是官方给的版本对应关系。这里选择一个版本举例：创建父工程pom配置如下：

```xml
<properties>
   <spring.boot.version>2.2.5.RELEASE</spring.boot.version>
   <spring.cloud.alibaba.version>2.2.1.RELEASE</spring.cloud.alibaba.version>
   <spring.cloud.version>Hoxton.SR3</spring.cloud.version>
</properties>
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring.cloud.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>com.alibaba.cloud</groupId>
      <artifactId>spring-cloud-alibaba-dependencies</artifactId>
      <version>${spring.cloud.alibaba.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring.boot.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```



## nacos作为注册中心

> 将服务注册到nacos注册中心，可以有效的管理服务之间的通信问题。开发比较简单,就是改造代码，将你的服务作为client客户端注册到nacos server注册中心。这里有两点注意：
>
> - 搭建好nacos注册中心服务（单机或者集群）
> - 修改服务引入依赖

### 引入nacos客户端客户端依赖

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

### 启动类注解@EnableDiscoveryClient

> @EnableDiscoveryClient标注在启动类上面： 表示开启服务发现与注册机制，向注册中心注册服务

```java
package com.itmck.product;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;


@EnableDiscoveryClient  //当前位置就是新增的注解，表示作为客户端向注册中心注册
@SpringBootApplication
public class ProductSpringBoot {

    public static void main(String[] args) {
        SpringApplication.run(ProductSpringBoot.class, args);
    }
}
```

### 配置文件

```yml
spring:
  application:
    name: spring-cloud-product  #当前应用名
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848 #注册中心地址,集群以逗号隔开
```

### 验证是否注册成功

![image-20230506102948516.png](https://s2.loli.net/2023/05/06/qHMGouz4AxvPKj9.png)

如图所示，表示注册成功。其中

- 服务名,默认：${spring.application.name}
- 分组名称默认：DEFAULT_GROUP
- 命名空间namespace默认：public
- 集群名默认：DEFAULT

服务在调用的时候，namespace+分组名+服务名

我们可以针对实际开发环境进行调整。比如可以将namespace改成dev开发环境，pro生产环境，将分组名称改为product商品组



## nacos作为配置中心

>nacos作为配置中心使用，可以将常用的配置信息直接放在配置中心，而不用每个服务都写一套。另外配置中心可以实时刷新服务的所需配置。同样使用nacos作为配置中心需要引入依赖，配置环境信息.

### 引入依赖

```xml
<dependency>
  <groupId>com.alibaba.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

### 配置yml

````yml
spring:
  application:
    name: spring-cloud-product  #当前应用名
  cloud:
    nacos:
      config:
        server-addr: localhost:8848 #配置中心地址,集群以逗号隔开
        file-extension: yaml
        namespace: bcf855b8-4d1a-4908-8f15-2c338cc538b9 # 配置命名空间
        group: product
        cluster-name: product-bj
````

如图所示其中spring.cloud.nacos.config 下为新增的配置。如下配置默认会先找配置中心的配置，如果存在就使用，不存在再找本地配置。整个使用规则遵循

namespace+group+配置文件名（默认spring.application.name）

### 配置环境信息

![image-20230506103033191.png](https://s2.loli.net/2023/05/06/GfuswvApVYtFW6L.png)

![image-20230506103041018.png](https://s2.loli.net/2023/05/06/irvXSlYwCgD1ZLF.png)



### 总结

> 在做微服务开发的时候，通常情况下都是将nacos作为配置中心与注册中心使用，所以在创建客户端的时候，直接将其依赖全部引入

```xml
<dependencies>
  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
  </dependency>
  <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
  </dependency>
</dependencies>
```

yml配置如下：

```yml
spring:
  application:
    name: springcloud-product  #当前应用名
  cloud:
    nacos:
      discovery:
        server-addr: 81.69.8.183:8848 #注册中心地址,集群以逗号隔开
      config:
        server-addr: 81.69.8.183:8848 #配置中心地址,集群以逗号隔开
        file-extension: yaml
```

启动类标注注解`@EnableDiscoveryClient`将服务作为客户端注册到nacos



## nacos注册中心源码分析

> 学习了上面的nacos作为注册中心与配置中心的基本使用之后，不难发现我们通过少量的配置加注解即可实现。那注解背后做了哪些事情。

我们知道nacos服务是作为独立的服务启动，而服务间的交互常见的有rpc，http，socket等。nacos使用的是哪个？

### 客户端如何进行注册？

> 通过spring-cloud-starter-alibaba-nacos-discovery找到spring.factories
>
> `这里为什么直接找spring.factories 需要对springboot的自动装配原理有一定的了解。这里不赘述`

![image-20230506103149850.png](https://s2.loli.net/2023/05/06/CwTmP7kQsYnxtUb.png)

看到图中标记，com.alibaba.cloud.nacos.registry.`NacosServiceRegistryAutoConfiguration` 看到名字不难发现，他就是自动注册配置类。

```java
@Configuration(
    proxyBeanMethods = false
)
@EnableConfigurationProperties
@ConditionalOnNacosDiscoveryEnabled
@ConditionalOnProperty(
    value = {"spring.cloud.service-registry.auto-registration.enabled"},
    matchIfMissing = true
)
@AutoConfigureAfter({AutoServiceRegistrationConfiguration.class, AutoServiceRegistrationAutoConfiguration.class, NacosDiscoveryAutoConfiguration.class})
public class NacosServiceRegistryAutoConfiguration {
    public NacosServiceRegistryAutoConfiguration() {
    }

    @Bean
    public NacosServiceRegistry nacosServiceRegistry(NacosDiscoveryProperties nacosDiscoveryProperties) {
        //这里就是向nacos注册
        return new NacosServiceRegistry(nacosDiscoveryProperties);
    }
  //...
  
  // 省略其他
}

```

进入代码不难发现注册方法

```java
public class NacosServiceRegistry implements ServiceRegistry<Registration> {

	@Override
	public void register(Registration registration) {

		if (StringUtils.isEmpty(registration.getServiceId())) {
			log.warn("No service to register for nacos client...");
			return;
		}

		String serviceId = registration.getServiceId();
		String group = nacosDiscoveryProperties.getGroup();

    //创建一个实例对象
		Instance instance = getNacosInstanceFromRegistration(registration);

		try {
      //向注册中心注册当前实例
			namingService.registerInstance(serviceId, group, instance);
			log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
					instance.getIp(), instance.getPort());
		}
		catch (Exception e) {
			log.error("nacos registry, {} register failed...{},", serviceId,
					registration.toString(), e);
		}
	}
  //创建注册实例，主机，端口，权重，机群名以及原数据信息
  private Instance getNacosInstanceFromRegistration(Registration registration) {
		Instance instance = new Instance();
		instance.setIp(registration.getHost());
		instance.setPort(registration.getPort());
		instance.setWeight(nacosDiscoveryProperties.getWeight());
		instance.setClusterName(nacosDiscoveryProperties.getClusterName());
		instance.setMetadata(registration.getMetadata());

		return instance;
	}
}

```

进入注册方法内`NacosNamingService`

```java
@SuppressWarnings("PMD.ServiceOrDaoClassShouldEndWithImplRule")
public class NacosNamingService implements NamingService {

@Override
 public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {

        if (instance.isEphemeral()) {
            BeatInfo beatInfo = new BeatInfo();
            beatInfo.setServiceName(NamingUtils.getGroupedName(serviceName, groupName));
            beatInfo.setIp(instance.getIp());
            beatInfo.setPort(instance.getPort());
            beatInfo.setCluster(instance.getClusterName());
            beatInfo.setWeight(instance.getWeight());
            beatInfo.setMetadata(instance.getMetadata());
            beatInfo.setScheduled(false);
            beatInfo.setPeriod(instance.getInstanceHeartBeatInterval());
						//心跳检测
            beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);
        }
				//注册实例，这里点进去就是向注册中心进行客户端实力的注册
        serverProxy.registerService(NamingUtils.getGroupedName(serviceName, groupName), groupName, instance);
    }
}

```

我们接着跟进代码 `NamingProxy`

```java
public class NamingProxy {

  public void registerService(String serviceName, String groupName, Instance instance) throws NacosException {

        NAMING_LOGGER.info("[REGISTER-SERVICE] {} registering service {} with instance: {}",
            namespaceId, serviceName, instance);

        final Map<String, String> params = new HashMap<String, String>(9);
        params.put(CommonParams.NAMESPACE_ID, namespaceId);
        params.put(CommonParams.SERVICE_NAME, serviceName);
        params.put(CommonParams.GROUP_NAME, groupName);
        params.put(CommonParams.CLUSTER_NAME, instance.getClusterName());
        params.put("ip", instance.getIp());
        params.put("port", String.valueOf(instance.getPort()));
        params.put("weight", String.valueOf(instance.getWeight()));
        params.put("enable", String.valueOf(instance.isEnabled()));
        params.put("healthy", String.valueOf(instance.isHealthy()));
        params.put("ephemeral", String.valueOf(instance.isEphemeral()));
        params.put("metadata", JSON.toJSONString(instance.getMetadata()));
				//调用http向nacos注册中心发出post注册请求
        reqAPI(UtilAndComs.NACOS_URL_INSTANCE, params, HttpMethod.POST);
    }
}


```

注意这个reqAPI中的`UtilAndComs.NACOS_URL_INSTANCE`,可以看到UtilAndComs.NACOS_URL_INSTANCE, params请求得地址拼接之后就是 `/nacos/v1/ns/instance`

![image-20230506103231062.png](https://s2.loli.net/2023/05/06/ljdkzemVCbS3q78.png)

**总结**

> 客户端引入依赖配置启动注解之后，在服务启动的时候就会通过http发送post请求，将当前服务实例的原数据等信息推送到nacos注册中心。
>
> 其中调用的地址就是 http:ip:port/nacos/v1/ns/instance.  [ip：注册中心地址，port：端口]

我们通过nacos官网查看 https://nacos.io/zh-cn/docs/open-api.html 通过官方API可以得到印证,与我们得到的一致.

![image-20230506103240806.png](https://s2.loli.net/2023/05/06/dnWTZPEgNqIhmAb.png)



### 心跳检测

> 上面我们在查看注册实例源码的时候出现了心跳检测，这里叙述一下:

```java
@SuppressWarnings("PMD.ServiceOrDaoClassShouldEndWithImplRule")
public class NacosNamingService implements NamingService {

@Override
 public void registerInstance(String serviceName, String groupName, Instance instance) throws NacosException {

        if (instance.isEphemeral()) {
            BeatInfo beatInfo = new BeatInfo();
            beatInfo.setServiceName(NamingUtils.getGroupedName(serviceName, groupName));
            beatInfo.setIp(instance.getIp());
            beatInfo.setPort(instance.getPort());
            beatInfo.setCluster(instance.getClusterName());
            beatInfo.setWeight(instance.getWeight());
            beatInfo.setMetadata(instance.getMetadata());
            beatInfo.setScheduled(false);
            beatInfo.setPeriod(instance.getInstanceHeartBeatInterval());
						//心跳检测
            beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);
        }
				//注册实例，这里点进去就是向注册中心进行客户端实力的注册
        serverProxy.registerService(NamingUtils.getGroupedName(serviceName, groupName), groupName, instance);
    }
}

```

重点关注一下：`beatReactor.addBeatInfo(NamingUtils.getGroupedName(serviceName, groupName), beatInfo);`

```java
    public void addBeatInfo(String serviceName, BeatInfo beatInfo) {
        NAMING_LOGGER.info("[BEAT] adding beat: {} to beat map.", beatInfo);
        String key = buildKey(serviceName, beatInfo.getIp(), beatInfo.getPort());
        BeatInfo existBeat = null;
        //fix #1733
        if ((existBeat = dom2Beat.remove(key)) != null) {
            existBeat.setStopped(true);
        }
        dom2Beat.put(key, beatInfo);
        executorService.schedule(new BeatTask(beatInfo), beatInfo.getPeriod(), TimeUnit.MILLISECONDS);//延迟进行心跳注册
        MetricsMonitor.getDom2BeatSizeMonitor().set(dom2Beat.size());
    }
```

从源码可以看到  executorService.schedule(`new BeatTask(beatInfo)`, beatInfo.getPeriod(), TimeUnit.MILLISECONDS);//延迟进行心跳注册

```java
class BeatTask implements Runnable {

        BeatInfo beatInfo;

        public BeatTask(BeatInfo beatInfo) {
            this.beatInfo = beatInfo;
        }

        @Override
        public void run() {
          	//如果是服务停止则结束
            if (beatInfo.isStopped()) {
                return;
            }
            long nextTime = beatInfo.getPeriod();//获取下次执行时间
            try {
                //发送心跳
                JSONObject result = serverProxy.sendBeat(beatInfo, BeatReactor.this.lightBeatEnabled);
                long interval = result.getIntValue("clientBeatInterval");
                boolean lightBeatEnabled = false;
                if (result.containsKey(CommonParams.LIGHT_BEAT_ENABLED)) {
                    lightBeatEnabled = result.getBooleanValue(CommonParams.LIGHT_BEAT_ENABLED);
                }
                BeatReactor.this.lightBeatEnabled = lightBeatEnabled;
                if (interval > 0) {
                    nextTime = interval;
                }
                int code = NamingResponseCode.OK;
                if (result.containsKey(CommonParams.CODE)) {
                    code = result.getIntValue(CommonParams.CODE);
                }
              	//如果不是成功状态，则从新注册实例到注册中心
                if (code == NamingResponseCode.RESOURCE_NOT_FOUND) {
                    Instance instance = new Instance();
                    instance.setPort(beatInfo.getPort());
                    instance.setIp(beatInfo.getIp());
                    instance.setWeight(beatInfo.getWeight());
                    instance.setMetadata(beatInfo.getMetadata());
                    instance.setClusterName(beatInfo.getCluster());
                    instance.setServiceName(beatInfo.getServiceName());
                    instance.setInstanceId(instance.getInstanceId());
                    instance.setEphemeral(true);
                    try {
                        serverProxy.registerService(beatInfo.getServiceName(),NamingUtils.getGroupName(beatInfo.getServiceName()), instance);
                    } catch (Exception ignore) {
                    }
                }
            } catch (NacosException ne) {
                NAMING_LOGGER.error("[CLIENT-BEAT] failed to send beat: {}, code: {}, msg: {}",
                    JSON.toJSONString(beatInfo), ne.getErrCode(), ne.getErrMsg());

            }
           //延迟接着从新执行
            executorService.schedule(new BeatTask(beatInfo), nextTime, TimeUnit.MILLISECONDS);
        }
    }
```

**发送心跳源码sendBeat**

```java
    public JSONObject sendBeat(BeatInfo beatInfo, boolean lightBeatEnabled) throws NacosException {

        if (NAMING_LOGGER.isDebugEnabled()) {
            NAMING_LOGGER.debug("[BEAT] {} sending beat to server: {}", namespaceId, beatInfo.toString());
        }
        Map<String, String> params = new HashMap<String, String>(8);
        String body = StringUtils.EMPTY;
        if (!lightBeatEnabled) {
            try {
                body = "beat=" + URLEncoder.encode(JSON.toJSONString(beatInfo), "UTF-8");
            } catch (UnsupportedEncodingException e) {
                throw new NacosException(NacosException.SERVER_ERROR, "encode beatInfo error", e);
            }
        }
        params.put(CommonParams.NAMESPACE_ID, namespaceId);
        params.put(CommonParams.SERVICE_NAME, beatInfo.getServiceName());
        params.put(CommonParams.CLUSTER_NAME, beatInfo.getCluster());
        params.put("ip", beatInfo.getIp());
        params.put("port", String.valueOf(beatInfo.getPort()));
      	//通过http调用nacos注册中心接口，注册上报心跳
        String result = reqAPI(UtilAndComs.NACOS_URL_BASE + "/instance/beat", params, body, HttpMethod.PUT);
        return JSON.parseObject(result);
    }
```

### 客户端获取注册中心实例列表

> 作为客户端要获取注册中心的实例列表，便于后续的ribbon负载均衡的调用，那他是如何获的

源码进去找到 NacosNamingService在他的源码中有服务注册，服务实例的获取

```java
@Override
public List<Instance> getAllInstances(String serviceName, String groupName, List<String> clusters, boolean subscribe) throws NacosException {

  ServiceInfo serviceInfo;
  if (subscribe) {
    //从缓存中获取服务实例，如果缓存为空调用远程接口获取服务实例列表
    serviceInfo = hostReactor.getServiceInfo(NamingUtils.getGroupedName(serviceName, groupName), StringUtils.join(clusters, ","));
  } else {
    //直接远程获取实例列表
    serviceInfo = hostReactor.getServiceInfoDirectlyFromServer(NamingUtils.getGroupedName(serviceName, groupName), StringUtils.join(clusters, ","));
  }
  List<Instance> list;
  if (serviceInfo == null || CollectionUtils.isEmpty(list = serviceInfo.getHosts())) {
    return new ArrayList<Instance>();
  }
  return list;
}
```

**hostReactor.getServiceInfo**

```java
    public ServiceInfo getServiceInfo(final String serviceName, final String clusters) {

        NAMING_LOGGER.debug("failover-mode: " + failoverReactor.isFailoverSwitch());
        String key = ServiceInfo.getKey(serviceName, clusters);
        if (failoverReactor.isFailoverSwitch()) {
            return failoverReactor.getService(key);
        }
				//获取客户端的服务实例缓存列表
        ServiceInfo serviceObj = getServiceInfo0(serviceName, clusters);

        if (null == serviceObj) {
            serviceObj = new ServiceInfo(serviceName, clusters);

            serviceInfoMap.put(serviceObj.getKey(), serviceObj);

            updatingMap.put(serviceName, new Object());
          	//如果缓存为空调用注册中心接口获取最新的服务实例列表
            updateServiceNow(serviceName, clusters);
            updatingMap.remove(serviceName);

        } else if (updatingMap.containsKey(serviceName)) {

            if (UPDATE_HOLD_INTERVAL > 0) {
                // hold a moment waiting for update finish
                synchronized (serviceObj) {
                    try {
                        serviceObj.wait(UPDATE_HOLD_INTERVAL);
                    } catch (InterruptedException e) {
                        NAMING_LOGGER.error("[getServiceInfo] serviceName:" + serviceName + ", clusters:" + clusters, e);
                    }
                }
            }
        }
				//执行延迟的定时任务,定时更新服务实例列表并更新本地map缓存
        scheduleUpdateIfAbsent(serviceName, clusters);

        return serviceInfoMap.get(serviceObj.getKey());
    }
```

**updatingMap.put(serviceName, new Object()) 方法如下：**

```java
    public void updateServiceNow(String serviceName, String clusters) {
        ServiceInfo oldService = getServiceInfo0(serviceName, clusters);
        try {
						//远程调用注册中心获取服务实例列表
            String result = serverProxy.queryList(serviceName, clusters, pushReceiver.getUDPPort(), false);

            if (StringUtils.isNotEmpty(result)) {
                processServiceJSON(result);
            }
        } catch (Exception e) {
            NAMING_LOGGER.error("[NA] failed to update serviceName: " + serviceName, e);
        } finally {
            if (oldService != null) {
                synchronized (oldService) {
                    oldService.notifyAll();
                }
            }
        }
    }
```

**com.alibaba.nacos.client.naming.net.NamingProxy#queryList**

```java
    public String queryList(String serviceName, String clusters, int udpPort, boolean healthyOnly)
        throws NacosException {

        final Map<String, String> params = new HashMap<String, String>(8);
        params.put(CommonParams.NAMESPACE_ID, namespaceId);
        params.put(CommonParams.SERVICE_NAME, serviceName);
        params.put("clusters", clusters);
        params.put("udpPort", String.valueOf(udpPort));
        params.put("clientIP", NetUtils.localIP());
        params.put("healthyOnly", String.valueOf(healthyOnly));
				//发送http请求获取服务实例列表
        return reqAPI(UtilAndComs.NACOS_URL_BASE + "/instance/list", params, HttpMethod.GET);
    }
```



**通过上面的心跳注册以及客户端注册，获取实例列表等源码不难看出，最终出口方法都在`NamingProxy`类中。**
