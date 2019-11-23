# Spring Cloud Config

## 写在前面

​	在我们的实际开发过程中，或多或少的应用到配置项，在分布式系统中，配置项更是重要的组成部分，在编辑配置过程中，出现了不方便维护、配置内容的安全与权限，更新配置项需要重启应用等诸多问题，这时候统一配置中心就出现了。

​	在Spring Cloud中，分布式配置中心组件Spring Cloud Config就是用做统一配置中心的。它支持配置文件放在在配置服务的内存中，也支持放在远程Git仓库里。引入Spring Cloud Config后，我们的外部配置文件就可以集中放置在一个git仓库里，再新建一个config server，用来管理所有的配置文件，维护的时候需要更改配置时，只需要在本地更改后，推送到远程仓库，所有的服务实例都可以通过config server来获取配置文件，这时每个服务实例就相当于配置服务的客户端config client,为了保证系统的稳定，配置服务端config server可以进行集群部署，即使某一个实例，因为某种原因不能提供服务，也还有其他的实例保证服务的继续进行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214112738657.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

## Config Server

### 使用方式

pom.xml文件中引入config-server依赖

```xml
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

启动类加入@EnableConfigServer注解

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableConfigServer
public class ConfigApplication {

   public static void main(String[] args) {
      SpringApplication.run(ConfigApplication.class, args);
   }
}
```

application.yml配置文件中配置远程git地址

```yaml
spring:
  application:
    name: config
  cloud:
    config:
      server:
        git:
          uri: https://gitlab-demo.com/SpringCloud_Sell/config-repo.git
          username: ********
          password: ********
```

### 资源文件与URL地址映射

Spring Cloud会将配置映射为"/{application}/{profile}"

URL地址和资源文件映射如下:

- /{application}/{profile}[/{label}]

- /{application}-{profile}.yml
- /{label}/{application}-{profile}.yml
- /{application}-{profile}.properties
- /{label}/{application}-{profile}.properties

1. 第一个规则的分支名是可以省略的，默认是master分支

2. 无论你的配置文件是properties，还是yml，只要是应用名+环境名能匹配到这个配置文件，那么就能取到
3. 如果是想直接定位到没有写环境名的默认配置，那么就可以使用default去匹配没有环境名的配置文件
4. 使用第一个规则会匹配到默认配置
5. 如果直接使用应用名来匹配，会出现404错误，此时可以加上分支名匹配到默认配置文件
6. 如果配置文件的命名很由多个-分隔，此时直接使用这个文件名去匹配的话，会出现直接将内容以源配置文件内容直接返回，内容前可能会有默认配置文件的内容
7. 如果文件名含有多个“-”，则以最后一个“-”分割{application}和{profile}，若文件名为：my-app-demo-dev.properties，则映射的url为"/my-app-demo/dev"
8. 示例：资源文件为myapp-dev.properties，对应url为:http://xxx/myapp/dev

####  客户端配置：

spring.application.name=xxxx

spring.cloud.config.profile=dev

spring.cloud.config.label=test

上述配置与资源文件对应关系为：

spring.application.name  对应  **{application}**

spring.cloud.config.profile  对应  **{profile}**

spring.cloud.config.label  对应  **{label}**

## Config Client

### 使用方式

pom.xml文件中引入config-server依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

添加 **bootstrap.yml** 应用配置文件，让spring boot启动时先加载bootstrap.yml去获取配置信息

```yaml
spring:
  application:
    name: order
  cloud:
    config:
      discovery:
        enabled: true
        service-id: CONFIG
      profile: test
      label: master #当 ConfigServer 的后端存储的是 Git 的时候，默认就是 master
```

**注意：**git上的配置文件中，order为公用的配置，每次都会加载，其他为区分不同环境的配置信息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214112812727.png)

## 自动更新配置

​	以上只能支持配置的基本使用，修改了git上的配置文件，还是必须要重启应用才能生效，没有达到统一配置中心的要求，那么如何才能自动更新配置呢？

### Spring Cloud Bus

  

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214112824611.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

看到上图是config的架构图，config-server会去远端的git拉取配置，拉取到配置后，会在本地存储一份，同时提供对外的配置服务，order，product等服务在启动的时候访问config-server读取配置，如果启动后再次修改git中的配置，order，product等服务的配置是不变的，原因是根本没有通知order，product等服务配置修改的消息。这时候就应用到了消息队列（rabbitMQ、kafka等），操作数据库就会应用到Spring Cloud Bus这个组件。config-server与config-client间就是通过消息队列进行通信，config-server使用了Spring Cloud Bus以后，会对外提供一个接口/bus-refresh，访问这个接口，就会把更新配置的信息发送到MQ中去。所以，只要在远端的git上配置好这样的地址就可以达到自动访问，自动刷新配置了。

注：git自动访问链接用到web-hooks功能

#### 使用方式

config-server与config-client的pom.xml文件中加入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
```

修改config-server配置文件，将/bus-refresh接口暴露出去

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214112847121.png)

在web-hooks上配置自动访问链接，这里不用配置成bus-refresh，config针对webhooks提供了一个路由monitor

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214112858496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

启动应用config-server与config-client后，会分别自动创建一个消息队列，当修改git上的配置后，webhooks自动访问了刷新配置的链接，这时候，队列中就会存在一个消息。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190214112908557.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

