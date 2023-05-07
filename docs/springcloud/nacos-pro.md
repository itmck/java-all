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