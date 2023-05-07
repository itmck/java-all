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

