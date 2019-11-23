---
tags: 
- 分布式微服务
---
### 简介

​	Spring Cloud Ribbon是一个基于HTTP和TCP的客户端负载均衡工具，它基于Netflix Ribbon实现。通过Spring Cloud的封装，可以让我们轻松地将面向服务的REST模版请求自动转换成客户端负载均衡的服务调用。Spring Cloud Ribbon虽然只是一个工具类框架，它不像服务注册中心、配置中心、API网关那样需要独立部署，但是它几乎存在于每一个Spring Cloud构建的微服务和基础设施中。因为微服务间的调用，API网关的请求转发等内容，实际上都是通过Ribbon来实现的，包括后续我们将要介绍的Feign，它也是基于Ribbon实现的工具。所以，对Spring Cloud Ribbon的理解和使用，对于我们使用Spring Cloud来构建微服务非常重要。Ribbon默认为我们提供了很多负载均衡算法，例如轮询、随机等。当然，我们也可为Ribbon实现自定义的负载均衡算法。

Ribbon的负载均衡应用在以下几方面：

- RestTemplate
- Feign
- Zuul

Ribbon实现软负载均衡有三点：

- 服务发现

  发现依赖服务的列表，依据服务的名字，把该服务所有的实例找到。

- 服务选择规则

  依据规则策略，如何从多个服务中选择一个有效的服务

- 服务监听

  检测失效的服务，做到高效剔除

Ribbon的主要组件包括：

- ServerList
- IRule
- ServerListFilter

​	首先通过ServerList获取所有可用服务列表，然后通过ServerListFilter过滤掉一部分地址，最后剩下的地址中，使用IRule选择一个实例作为最终目标结果。

负载均衡有好几种实现策略，常见的有：

1. 随机 (Random)
2. 轮询 (RoundRobin)
3. 一致性哈希 (ConsistentHash)
4. 哈希 (Hash)
5. 加权（Weighted）

### ILoadBalance 负载均衡器

​	Ribbon内部提供了一个叫做ILoadBalance的接口代表负载均衡器的操作，其中包括**添加服务器操作**、**选择服务器操作**、**获取所有的服务器列表**、**获取可用的服务器列表**等。

ILoadBalance的继承关系如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190203111736326.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

​	负载均衡器是从EurekaClient（EurekaClient的实现类为DiscoveryClient）获取服务信息，根据IRule去路由，并且根据IPing判断服务的可用性。在BaseLoadBalancer类下，BaseLoadBalancer的构造函数，该构造函数开启了一个PingTask任务setupPingTask();，代码如下：

```java
public BaseLoadBalancer(String name, IRule rule, LoadBalancerStats stats,
            IPing ping, IPingStrategy pingStrategy) {
    logger.debug("LoadBalancer [{}]:  initialized", name);

    this.name = name;
    this.ping = ping;
    this.pingStrategy = pingStrategy;
    setRule(rule);
    setupPingTask();
    lbStats = stats;
    init();
}
```

setupPingTask方法开启了ShutdownEnabledTimer执行PingTask任务，在默认情况下pingIntervalSeconds为10，即每10秒钟，向EurekaClient发送一次”ping”，并有可能从Eureka Client获取注册信息。针对PingTask，它根据pingerStrategy.pingServers(ping, allServers)来获取服务的可用性，如果该返回结果，如之前相同，则不去向EurekaClient获取注册列表，如果不同则通知ServerStatusChangeListener或者changeListeners发生了改变，进行更新或者重新拉取。

```java
class PingTask extends TimerTask {
    public void run() {
        try {
            new Pinger(pingStrategy).runPinger();
        } catch (Exception e) {
            logger.error("LoadBalancer [{}]: Error pinging", name, e);
        }
    }
}
```



```java
void setupPingTask() {
    if (canSkipPing()) {
        return;
    }
    if (lbTimer != null) {
        lbTimer.cancel();
    }
    lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,true);
    lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
    forceQuickPing();
}
```

如果可用性改变后，重新拉取了注册中心的数据，LoadBalancerClient有了这些服务注册的新列表，就可以根据具体的IRule来进行负载均衡。

### IRule 路由

IRule接口代表负载均衡策略：

```java
public interface IRule{
    /*
     * choose one alive server from lb.allServers or
     * lb.upServers according to key
     * 
     * @return choosen Server object. NULL is returned if none
     *  server is available 
     */

    public Server choose(Object key);
    
    public void setLoadBalancer(ILoadBalancer lb);
    
    public ILoadBalancer getLoadBalancer();    
}
```

IRule的实现关系如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190203111751777.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

### 自定义负责均衡策略

| 策略类                    | 命名             | 说明                                                         |
| ------------------------- | ---------------- | ------------------------------------------------------------ |
| RandomRule                | 随机策略         | 随机选择 Server                                              |
| RoundRobinRule            | 轮训策略         | 按顺序循环选择 Server                                        |
| RetryRule                 | 重试策略         | 在一个配置时问段内当选择 Server 不成功，则一直尝试选择一个可用的 Server |
| BestAvailableRule         | 最低并发策略     | 逐个考察 Server，如果 Server 断路器打开，则忽略，再选择其中并发连接最低的 Server |
| AvailabilityFilteringRule | 可用过滤策略     | 过滤掉一直连接失败并被标记为 `circuit tripped` 的 Server，过滤掉那些高并发连接的 Server（active connections 超过配置的网值） |
| ResponseTimeWeightedRule  | 响应时间加权策略 | 根据 Server 的响应时间分配权重。响应时间越长，权重越低，被选择到的概率就越低；响应时间越短，权重越高，被选择到的概率就越高。这个策略很贴切，综合了各种因素，如：网络、磁盘、IO等，这些因素直接影响着响应时间 |
| ZoneAvoidanceRule         | 区域权衡策略     | 综合判断 Server 所在区域的性能和 Server 的可用性轮询选择 Server，并且判定一个 AWS Zone 的运行性能是否可用，剔除不可用的 Zone 中的所有 Server |

#### 全局配置

- ##### 配置类方式

```java
@Configuration
public class RibbonGlobalLoadBalancingConfiguration {
    /**
     * 随机规则
     */
    @Bean
    public IRule ribbonRule() {
        return new RandomRule();
    }
}
```

#### 指定服务配置

- ##### 注解方式

增加一个针对单个服务的 Ribbon 负载聚恒策略配置类：

```java
@Configuration
//标记使用的注解
@AvoidScan
public class RibbonRandomLoadBalancingConfiguration {

    //针对客户端的配置管理器。
    @Resource
    IClientConfig clientConfig;

    @Bean
    public IRule ribbonRule(IClientConfig clientConfig) {
        return new RandomRule();
    }

}
```

启动类上加入针对单个服务的负载均衡策略：

```java
/** 配置针对单个服务的 Ribbon 负载均衡策略 **/
@RibbonClient(
    name = "users", configuration = RibbonRandomLoadBalancingConfiguration.class
)
/** 此处配置根据标识 @AvoidScan 过滤掉需要单独配置的 Ribbon 负载均衡策略，不然就会作用于全局，启动就会报错 */
@ComponentScan(
        excludeFilters = @ComponentScan.Filter(
                type = FilterType.ANNOTATION, value = AvoidScan.class
        )
)
public class UsersApplication {

	public static void main(String[] args) {
		SpringApplication.run(UsersApplication.class, args);
	}
}
```

- ##### 配置文件方式（推荐）

**application.yml.** 

```yaml
users:#服务名称
  ribbon:
    NIWSServerListClassName: com.netflix.loadbalancer.ConfigurationBasedServerList
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule
```

#### 














