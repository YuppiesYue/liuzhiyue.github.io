### 简介

​	Eureka 是 Netflix 出品的用于实现服务注册和发现的工具。 Spring Cloud 集成了 Eureka，并提供了开箱即用的支持。其中， Eureka 又可细分为 Eureka Server 和 Eureka Client。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190131200208127.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)
其中包含两个组件 ：

- Eureka Server **注册中心（服务端）**

  供服务注册的服务器，各个节点启动后，在Eureka Server中进行注册；

- Eureka Client **服务注册（客户端）** 

  Eureka Client 是一个Java客户端，用于和服务端进行交互，同时客户端也是一个内置的默认使用轮询负载均衡算法的负载均衡器。在应用启动后，会向Eureka Server发送心跳（默认30秒）。如果Eureka Server在多个心跳周期内没有接收到某个节点的心跳，Eureka Server将会从服务注册表中将这个服务移除（默认90秒）。

### Eureka与Zookeeper比较

​	著名的CAP理论指出，一个分布式系统不可能同时满足C(一致性)、A(可用性)和P(分区容错性)。由于分区容错性在是分布式系统中必须要保证的，因此我们只能在A和C之间进行权衡。在此Zookeeper保证的是CP, 而Eureka则是AP。

#### Zookeeper保证CP

​	当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟以前的注册信息，但不能接受服务直接down掉不可用。也就是说，服务注册功能对可用性的要求要高于一致性。但是zk会出现这样一种情况，当master节点因为网络故障与其他节点失去联系时，剩余节点会重新进行leader选举。问题在于，选举leader的时间太长，30 ~ 120s, 且选举期间整个zk集群都是不可用的，这就导致在选举期间注册服务瘫痪。在云部署的环境下，因网络问题使得zk集群失去master节点是较大概率会发生的事，虽然服务能够最终恢复，但是漫长的选举时间导致的注册长期不可用是不能容忍的。

#### Eureka保证AP

​	Eureka看明白了这一点，因此在设计时就优先保证可用性。Eureka各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和查询服务。而Eureka的客户端在向某个Eureka注册或时如果发现连接失败，则会自动切换至其它节点，只要有一台Eureka还在，就能保证注册服务可用(保证可用性)，只不过查到的信息可能不是最新的(不保证强一致性)。除此之外，Eureka还有一种自我保护机制，如果在15分钟内超过85%的节点都没有正常的心跳，那么Eureka就认为客户端与注册中心出现了网络故障，此时会出现以下几种情况： 

1. Eureka不再从注册列表中移除因为长时间没收到心跳而应该过期的服务 
2. Eureka仍然能够接受新服务的注册和查询请求，但是不会被同步到其它节点上(即保证当前节点依然可用) 
3. 当网络稳定时，当前实例新的注册信息会被同步到其它节点中

​	因此， Eureka可以很好的应对因网络故障导致部分节点失去联系的情况，而不会像zookeeper那样使整个注册服务瘫痪。Eureka作为单纯的服务注册中心来说要比zookeeper更加“专业”，因为注册服务更重要的是可用性，我们可以接受短期内达不到一致性的状况。不过Eureka目前1.X版本的实现是基于servlet的[Java ](http://lib.csdn.net/base/java)web应用，它的极限性能肯定会受到影响。期待正在开发之中的2.X版本能够从servlet中独立出来成为单独可部署执行的服务。

### Eureka Server端的使用

在介绍Eureka Server端的使用之前，先来看一下**Spring Cloud**与**Spring Boot**的版本兼容问题。

以下两个表格出自Spring官网（http://spring.io/projects/spring-cloud）

| Release Train | Boot Version |
| ------------- | ------------ |
| Greenwich     | 2.1.x        |
| Finchley      | 2.0.x        |
| Edgware       | 1.5.x        |
| Dalston       | 1.5.x        |

| Component                 | Edgware.SR5    | Finchley.SR2  | Finchley.BUILD-SNAPSHOT |
| ------------------------- | -------------- | ------------- | ----------------------- |
| spring-cloud-aws          | 1.2.3.RELEASE  | 2.0.1.RELEASE | 2.0.1.BUILD-SNAPSHOT    |
| spring-cloud-bus          | 1.3.3.RELEASE  | 2.0.0.RELEASE | 2.0.1.BUILD-SNAPSHOT    |
| spring-cloud-cli          | 1.4.1.RELEASE  | 2.0.0.RELEASE | 2.0.1.BUILD-SNAPSHOT    |
| spring-cloud-commons      | 1.3.5.RELEASE  | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-contract     | 1.2.6.RELEASE  | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-config       | 1.4.5.RELEASE  | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-netflix      | 1.4.6.RELEASE  | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-security     | 1.2.3.RELEASE  | 2.0.1.RELEASE | 2.0.1.BUILD-SNAPSHOT    |
| spring-cloud-cloudfoundry | 1.1.2.RELEASE  | 2.0.1.RELEASE | 2.0.1.BUILD-SNAPSHOT    |
| spring-cloud-consul       | 1.3.5.RELEASE  | 2.0.1.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-sleuth       | 1.3.5.RELEASE  | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-stream       | Ditmars.SR4    | Elmhurst.SR1  | Elmhurst.BUILD-SNAPSHOT |
| spring-cloud-zookeeper    | 1.2.2.RELEASE  | 2.0.0.RELEASE | 2.0.1.BUILD-SNAPSHOT    |
| spring-boot               | 1.5.16.RELEASE | 2.0.6.RELEASE | 2.0.7.BUILD-SNAPSHOT    |
| spring-cloud-task         | 1.2.3.RELEASE  | 2.0.0.RELEASE | 2.0.1.BUILD-SNAPSHOT    |
| spring-cloud-vault        | 1.1.2.RELEASE  | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-gateway      | 1.0.2.RELEASE  | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-openfeign    |                | 2.0.2.RELEASE | 2.0.2.BUILD-SNAPSHOT    |
| spring-cloud-function     | 1.0.1.RELEASE  | 1.0.0.RELEASE | 1.0.1.BUILD-SNAPSHOT    |

这里服务端我们所用的version为

Spring Boot		**2.0.2.RELEASE**

Spring Cloud		**Finchley.RELEASE**

#### POM.xml

导入相依的依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

#### 主程序类

加入注解

**@EnableEurekaServer**

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {
	public static void main(String[] args) {
		SpringApplication.run(EurekaApplication.class, args);
	}
}
```

#### application.yml配置

```yml
eureka:
  client:
    #false表示不向注册中心注册自己。
    register-with-eureka: false 
    #false表示自己端就是注册中心，我的职责就是维护服务实例，并不需要去检索服务
    fetch-registry: false
    #Eureka高复用时设置其他的Eureka之间通信defaultZone配置多个地址
    service-url:
      defaultZone: http://localhost:8762/eureka/
  server:
  	#Eureka服务端关闭心跳连接测试
    enable-self-preservation: false 
spring:
  application:
    name: eureka
server:
  port: 8761
```

启动程序访问http://localhost:8761/即可看到Eureka控制台

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190131200345502.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)
### Eureka Client端的使用

#### POM.xml

导入相依的依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

#### 主程序类

加入注解：（注意和服务端区分开）

**@EnableDiscoveryClient** 

```java
@SpringBootApplication
@EnableDiscoveryClient
public class ClientApplication {
	public static void main(String[] args) {
		SpringApplication.run(ClientApplication.class, args);
	}
}
```

#### application.yml配置

```yaml
eureka:
  client:
    service-url:
      #集群
      defaultZone: http://localhost:8761/eureka/,http://localhost:8762/eureka/           instance:
      #自定义名称
      hostname: clientName
      #访问路径可以显示IP地址
      prefer-ip-address: true
spring:
  application:
    name: client
```

启动程序访问Eureka Server的http://localhost:8761/即可看到Eureka控制台中注册上了client

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190131200412604.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

### Eureka Server高可用

​	Eureka Server进行互相注册的方式来实现高可用的部署，所以我们只需要将Eureka Server配置其他可用的serviceUrl就能实现高可用部署。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190131200434544.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

### 服务发现的两种方式

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190131200452661.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

- 客户端发现（Eureka）

  ​	在客户端发现的的形势，优点为简单直接，不需要代理的介入，同时客户端知道所有可用的服务的实际地址，缺点是客户端需要实现一套自己的逻辑，比如A服务想要挑选指定的B服务进行调用，就需要实现自己的逻辑了。

- 服务端发现（Nginx、Zookeeper、Kubernetes）

  ​	在服务端发现的形势，优点为由于代理的介入，B服务和注册中心对A是不可见的。A服务只需要找代理发一个请求就好了
