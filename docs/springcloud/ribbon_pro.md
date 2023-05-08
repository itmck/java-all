## Ribbon源码分析

### 源码猜测

1. 客户端想要进行负载均衡,必须要有服务端的地址列表信息(这里地址列表需要通过心跳定期维护)
2. 拿到地址列表信息通过负载均衡算法进行选择每次请求要调用的服务地址
3. 通过http进行请求server端

### 源码验证

1. client客户端对server服务端地址列表如何维护
2. client客户端对每次server服务端的时候rule的选择
3. 请求如何通过http进行

###  验证ribbon负载均衡

> 通过**spring-cloud-starter-alibaba-nacos-discovery** 下**META-INF\spring.factories**

![image-20230506170006294.png](https://s2.loli.net/2023/05/08/PYINCZHxLMRgzsr.png)

进入源码

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@ConditionalOnBean(SpringClientFactory.class)
@ConditionalOnRibbonNacos
@ConditionalOnNacosDiscoveryEnabled
@AutoConfigureAfter(RibbonAutoConfiguration.class)//这里就是ribbon自动配置类
@RibbonClients(defaultConfiguration = NacosRibbonClientConfiguration.class)
public class RibbonNacosAutoConfiguration {

}

```

进入org.springframework.cloud.netflix.ribbon.`RibbonAutoConfiguration`

```java
@Configuration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)
@RibbonClients
@AutoConfigureAfter(
		name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,   //进入当前负载均衡自动配置类
		AsyncLoadBalancerAutoConfiguration.class })
@EnableConfigurationProperties({ RibbonEagerLoadProperties.class,
		ServerIntrospectorProperties.class })
public class RibbonAutoConfiguration {
  //.....省略
}
```



```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

  //将标记有@LoadBalanced注解的RestTemplate注入
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();


	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {

		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
      //初始化负载均衡拦截器
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}
    
    @Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(
						restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
        //针对RestTemplate添加拦截器,拦截后做一些初始化操作
				restTemplate.setInterceptors(list);
			};
		}


	}
  //省略。。。。
}

```



进入到 LoadBalancerInterceptor 拦截器里面可以看到一个重写的方法`intercept`

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	private LoadBalancerClient loadBalancer;

	private LoadBalancerRequestFactory requestFactory;

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer,
			LoadBalancerRequestFactory requestFactory) {
		this.loadBalancer = loadBalancer;
		this.requestFactory = requestFactory;
	}

	public LoadBalancerInterceptor(LoadBalancerClient loadBalancer) {
		// for backwards compatibility
		this(loadBalancer, new LoadBalancerRequestFactory(loadBalancer));
	}

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null,"Request URI does not contain a valid hostname: " + originalUri);
    //关键点：通过负载均衡策略拿到服务实例地址，进行调用
		return this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
	}

}

```



org.springframework.cloud.netflix.ribbon.RibbonLoadBalancerClient#**`execute()`**

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {	
  
  //
  public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    //获取负载均衡器
      ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    //获取服务端地址
      Server server = getServer(loadBalancer, hint);
      if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
      }
      RibbonServer ribbonServer = new RibbonServer(serviceId, server,isSecure(server, serviceId),serverIntrospector(serviceId).getMetadata(server));

    //请求
      return execute(serviceId, ribbonServer, request);
    }
}
```



通过代码跟进 ILoadBalancer loadBalancer = getLoadBalancer(serviceId);可知ILoadBalancer 是从容器中获取的.那么说明当前ILoadBalancer 肯定是在容器启动的时候进行初始化了.

```java
	public <T> T getInstance(String name, Class<T> type) {
		AnnotationConfigApplicationContext context = getContext(name);
		if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
				type).length > 0) {
			return context.getBean(type);//这里负载均衡器是从容器中获取，说明已经初始化完成了，那他是在哪里完成的？？？？
		}
		return null;
	}

```

通过 org.springframework.cloud.netflix.ribbon.**RibbonClientConfiguration 可知.ILoadBalancer默认是**ZoneAwareLoadBalancer

```java
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
    //默认负载均衡器
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,serverListFilter, serverListUpdater);
	}
```

回到刚才位置可知

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient {	
  
  //
  public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    //获取负载均衡器
      ILoadBalancer loadBalancer = getLoadBalancer(serviceId);//ZoneAwareLoadBalancer
    //获取服务端地址
      Server server = getServer(loadBalancer, hint);
      if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
      }
      RibbonServer ribbonServer = new RibbonServer(serviceId, server,isSecure(server, serviceId),serverIntrospector(serviceId).getMetadata(server));

    //请求
      return execute(serviceId, ribbonServer, request);
    }
}
```



```java
	protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
		if (loadBalancer == null) {
			return null;
		}
		
    //负载均衡获取一个服务实例
		return loadBalancer.chooseServer(hint != null ? hint : "default");
	}
```

com.netflix.loadbalancer.BaseLoadBalancer#chooseServer

```java
public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
              //根据负载均衡策略选择一个服务实例
                return rule.choose(key);
            } catch (Exception e) {
                logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
                return null;
            }
        }
    }
```

rule初始化在org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration#ribbonRule

```java
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
    //默认负载均衡器
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,serverListFilter, serverListUpdater);
	}

```

通过上面可知,进入到com.netflix.loadbalancer.**PredicateBasedRule**

```java
public abstract class PredicateBasedRule extends ClientConfigEnabledRoundRobinRule {
   
    /**
     * Method that provides an instance of {@link AbstractServerPredicate} to be used by this class.
     * 
     */
    public abstract AbstractServerPredicate getPredicate();
        
    /**
     * Get a server by calling {@link AbstractServerPredicate#chooseRandomlyAfterFiltering(java.util.List, Object)}.
     * The performance for this method is O(n) where n is number of servers to be filtered.
     */
    @Override
    public Server choose(Object key) {
        ILoadBalancer lb = getLoadBalancer();
      //lb.getAllServers() 就是获取服务实例列表
        Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
        if (server.isPresent()) {
            return server.get();
        } else {
            return null;
        }       
    }
}

```

这里获取实例是从list缓存中获取，这也说明了一个情况，这个缓存肯定是在启动完成就初始化了数据的。

```java
public class BaseLoadBalancer extends AbstractLoadBalancer implements PrimeConnections.PrimeConnectionListener, IClientConfigAware {

   	//存放服务实例的列表
    protected volatile List<Server> allServerList = Collections.synchronizedList(new ArrayList<Server>());
    
    @Override
    public List<Server> getAllServers() {
      	//获取所有的实例
        return Collections.unmodifiableList(allServerList);
    }
   
}

```

com.netflix.loadbalancer.AbstractServerPredicate#chooseRoundRobinAfterFiltering()

```java
    public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
        List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
        if (eligible.size() == 0) {
            return Optional.absent();
        }
        return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
    }
```

关键点.**通过下面可知采用轮询方式进行选则服务**

解析:

modulo=1 return 0

modulo=2 return 0 ,1

modulo=3 return 0 ,1 ,2

AtomicInteger类compareAndSet通过原子操作实现了CAS操作，最底层基于汇编语言实现

compareAndSet(expect, update)先进行比较，如果相比较的两个值是相等的，那么就进行更新操作

```java
    private int incrementAndGetModulo(int modulo) {
        for (;;) {
            int current = nextIndex.get();
            int next = (current + 1) % modulo;
            if (nextIndex.compareAndSet(current, next) && current < modulo)
                return current;
        }
    }
```

org.springframework.cloud.netflix.ribbon.RibbonClientConfiguration

```java
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
```

com.netflix.loadbalancer.DynamicServerListLoadBalancer#DynamicServerListLoadBalancer()

```java
    public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                         ServerList<T> serverList, ServerListFilter<T> filter,
                                         ServerListUpdater serverListUpdater) {
        super(clientConfig, rule, ping);
        this.serverListImpl = serverList;
        this.filter = filter;
        this.serverListUpdater = serverListUpdater;
        if (filter instanceof AbstractServerListFilter) {
            ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
        }
      //这里初始化就获取到了实例列表
        restOfInit(clientConfig);
    }

```

```java
    void restOfInit(IClientConfig clientConfig) {
        boolean primeConnection = this.isEnablePrimingConnections();
        // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
        this.setEnablePrimingConnections(false);
        enableAndInitLearnNewServersFeature();

      //关键点：更新服务列表
        updateListOfServers();
        if (primeConnection && this.getPrimeConnections() != null) {
            this.getPrimeConnections()
                    .primeConnections(getReachableServers());
        }
        this.setEnablePrimingConnections(primeConnection);
        LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
    }
    
```



```java
@VisibleForTesting
public void updateListOfServers() {
  List<T> servers = new ArrayList<T>();
  if (serverListImpl != null) {
    //如果服务列表为空，发起请求获取实例列表
    servers = serverListImpl.getUpdatedListOfServers();
  }
  //更新
  updateAllServerList(servers);
}
```



```java
    @Override
    public void setServersList(List lsrv) {
        super.setServersList(lsrv);//这里就是把得到的列表赋值给
        List<T> serverList = (List<T>) lsrv;
        Map<String, List<Server>> serversInZones = new HashMap<String, List<Server>>();
        for (Server server : serverList) {
            // make sure ServerStats is created to avoid creating them on hot
            // path
            getLoadBalancerStats().getSingleServerStat(server);
            String zone = server.getZone();
            if (zone != null) {
                zone = zone.toLowerCase();
                List<Server> servers = serversInZones.get(zone);
                if (servers == null) {
                    servers = new ArrayList<Server>();
                    serversInZones.put(zone, servers);
                }
                servers.add(server);
            }
        }
        setServerListForZones(serversInZones);
    }
```

com.netflix.loadbalancer.BaseLoadBalancer#setServersList

```java
public class BaseLoadBalancer extends AbstractLoadBalancer implements
        PrimeConnections.PrimeConnectionListener, IClientConfigAware {
  
   	//存放服务实例的列表
    protected volatile List<Server> allServerList = Collections.synchronizedList(new ArrayList<Server>());

  
  	//设置服务实例列表到缓存中
		public void setServersList(List lsrv) {
        Lock writeLock = allServerLock.writeLock();
      
        ArrayList<Server> newServers = new ArrayList<Server>();
        writeLock.lock();
        try {
          //先创建一个list将实例存放进去，然后将当前列表赋值给allServerList。copyonwrite思想
            ArrayList<Server> allServers = new ArrayList<Server>();
            for (Object server : lsrv) {
                if (server == null) {
                    continue;
                }

                if (server instanceof String) {
                    server = new Server((String) server);
                }

                if (server instanceof Server) {
                    logger.debug("LoadBalancer [{}]:  addServer [{}]", name, ((Server) server).getId());
                  //
                    allServers.add((Server) server);
                } else {
                    throw new IllegalArgumentException(
                            "Type String or Server expected, instead found:"
                                    + server.getClass());
                }
            }

					//关键点就是这里，把服务列表赋值给了List<Server> allServerList
            allServerList = allServers;
            
        } finally {
            writeLock.unlock();
        }
    }
}
```



**验证通过http调用**

通过**restTemplate.getForObject()**进入不难发现接口通过http进行调用 org.springframework.web.client.RestTemplate#getForObject(...)

```java
	@Override
	@Nullable
	public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws RestClientException {
		RequestCallback requestCallback = acceptHeaderRequestCallback(responseType);
		HttpMessageConverterExtractor<T> responseExtractor =
				new HttpMessageConverterExtractor<>(responseType, getMessageConverters(), logger);
    //这里就是通过http发送请求
		return execute(url, HttpMethod.GET, requestCallback, responseExtractor, uriVariables);
	}

```

