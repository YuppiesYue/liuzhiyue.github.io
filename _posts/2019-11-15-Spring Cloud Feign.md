---
tags: 
- 分布式微服务
---
## Feign简介

​	Feign 是一个声明web服务客户端，在[RestTemplate](https://blog.csdn.net/javaliuzhiyue/article/details/86741932)的基础上对其封装，由它来帮助我们定义和实现依赖服务接口的定义。Spring Cloud Feign 基于Netflix Feign 实现的，整理Spring Cloud Ribbon 与 Spring Cloud Hystrix，并且实现了声明式的Web服务客户端定义方式。

## Feign使用

第一步：pom.xml文件中加入相关依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

第二步：启动类上加入注解@EnableFeignClients

```java
@EnableFeignClients
@SpringCloudApplication
public class OrderApplication {

   public static void main(String[] args) {
      SpringApplication.run(OrderApplication.class, args);
   }
}
```

第三步：创建客户端调用接口，声明要调用的方法

```java
@FeignClient(name = "product")
public interface ProductClient {

    @GetMapping("/msg")
    String productMsg();

    @PostMapping("/product/listForOrder")
    List<ProductInfo> listForOrder(@RequestBody List<String> productIdList);

}
```

第四步：直接注入接口进行调用（像调用本地方法一样）

```java
@RestController
@Slf4j
public class ClientController {

    @Autowired
    private ProductClient productClient;

    @GetMapping("/getProductMsg")
    public String getProductMsg() {

        String response = productClient.productMsg();
        log.info("response={}", response);
        return response;
    }

    @GetMapping("/getProductList")
    public String getProductList() {
        List<String> productIdList = Arrays.asList("1207")；
        List<ProductInfo> productInfoList = productClient.listForOrder(productIdList);
        log.info("response={}", productInfoList);
        return "ok";
    }
}
```

## 自定义feign配置

​	在SpringCloud 中默认的配置类是**FeignClientsConfiguration**,该类定义了Feign默认使用的编码器（feign.Decoder），解码器（feign.Encoder），所有使用的契约（feign.Contract）等。通过注解@FeignClient 的configuration属性来指定自定义配置类，由先级比FeignClientsConfiguration 高。

```java
@FeignClient(name = "stores", configuration = FooConfiguration.class)
public interface StoreClient {
    //..
}
```

配置类

```java
/**
 * 该类为Feign的配置类
 * 注意：该类不应该在主应用程序上下文的@CompantScan中,否则该类中的配置信息就会被所有的@FeignClient共享。
 */
@Configuration
public class FeignConfiguration {

    /**
     * 用feign.Contract.Default替换SpringMvcContract契约
     * Feign默认使用的契约是SpringMvcContract，因此它可以使用Spring MVC的注解。
     * @return
     */
    @Bean
    public Contract feignContract() {
        return new feign.Contract.Default();
    }

    /**
     * 日志级别配置
     *
     * NONE 不输出日志
     * BASIC 只有请求方法、URL、响应状态代码、执行时间
     * HEADERS基本信息以及请求和响应头
     * FULL 请求和响应 的heads、body、metadata
     *
     */
    @Bean
    Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;
    }

    /**
     * 设置 Eureka Server的访问的用户名和密码
     * @return
     */
    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor("user","123456");
    }

}
```
































