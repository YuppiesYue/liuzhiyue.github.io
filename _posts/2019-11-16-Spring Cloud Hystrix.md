# Spring Cloud Hystrix

## 写在前面

​	在微服务架构中，通常会有多个服务间互相调用，如果某个服务不可用，导致多个服务故障，造成整个系统不可用的情况被称为**雪崩效应**。Spring Cloud的防雪崩利器就是Hystrix，它是基于Netflix对应的Hystrix。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190219143123312.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

## 作用

- 服务降级

  所谓的服务降级是指，在调用一方服务的时候没有及时返回结果，或者调用失败等情况出现以后，系统将自动采取另外一只预备方案进行处理。优先核心服务，非核心服务不可用或弱可用。类似弃车保帅。

- 服务熔断
- 依赖隔离
- 监控（Hystrix Dashboard）

## 使用方式

### RestTemplate的形式

通过HystrixCommand注解指定，并在fallbackMethod（回退函数）中具体实现降级逻辑

在pom.xml中引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

在启动类中加入注解**@EnableCircuitBreaker**，但是在**@SpringCloudApplication**中已经包含了这个注解，所以只引入**@SpringCloudApplication**即可

```java
@EnableFeignClients
@SpringCloudApplication
public class OrderApplication {

   public static void main(String[] args) {
      SpringApplication.run(OrderApplication.class, args);
   }
}
```

```java
import com.imooc.order.enums.ResultEnum;
import com.imooc.order.exception.OrderException;
import com.netflix.hystrix.contrib.javanica.annotation.DefaultProperties;
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import java.util.Arrays;

@RestController
//默认的配置
@DefaultProperties(defaultFallback = "defaultFallback")
public class HystrixController {

    //超时配置
// @HystrixCommand(commandProperties = {
//       @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")
// })

    // @HystrixCommand(commandProperties = {
//       @HystrixProperty(name = "circuitBreaker.enabled", value = "true"),              //设置熔断
//       @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),    //请求数达到后才计算
//       @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000"), //休眠时间窗
//       @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60"),  //错误率到达60%后触发熔断
// })
    @HystrixCommand(fallbackMethod = "fallback")
    @GetMapping("/getProductInfoList")
    public String getProductInfoList(@RequestParam("number") Integer number) {
        if (number % 2 == 0) {
            return "success";
        }
        RestTemplate restTemplate = new RestTemplate();
        return restTemplate.postForObject("http://127.0.0.1:8005/product/listForOrder",
                Arrays.asList("1661622"),
                String.class);
    }

    //不指定具体的降级方法，则会进入默认的降级方法
    @HystrixCommand
    @GetMapping("/getProductInfo")
    public String getProductInfo() {
        throw new OrderException(ResultEnum.ORDER_NOT_EXIST);
    }

    private String fallback() {
        return "太拥挤了, 请稍后再试~~";
    }

    private String defaultFallback() {
        return "默认提示：太拥挤了, 请稍后再试~~";
    }
}
```

以上也可通过配置文件配置

```yaml
hystrix:
  command:
    default: #默认超时时间配置
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 1000
    getProductInfoList: #指定方法配置超时时间，此处配置成方法名
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 3000
```

### 熔断机制原理

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190219143201258.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

上图描述了断路器模式设计状态机，状态机有三种状态，closed（熔断器关闭）、open（熔断器开启）、half open（熔断器半开），当失败次数累计到一定的阈值，或者说一定的比例就会启动熔断机制，熔断器处于open状态的时候，此时对服务都直接返回错误，但设计了一个时钟选项，默认的时钟到了一定时间就会进入半熔断状态，允许定量的服务请求，如果调用都成功，或者一定的比例成功，则认为服务恢复正常，就会关闭熔断器，否则会认为依然服务异常，进入打开模式。

```java
//设置熔断
@HystrixProperty(name = "circuitBreaker.enabled", value = "true")
//请求数达到后才计算（达到10次后计算错误率）
@HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10")
//休眠时间窗
@HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000")
//错误率到达60%后触发熔断
@HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "60")
```

### feign-hystrix的使用

配置文件中加入

```yaml
feign:
  hystrix:
    enabled: true
```

feign接口中加入fallback属性

```java
import com.imooc.product.common.ProductInfoOutput;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import java.util.List;

//fallback用于配置回调类
@FeignClient(name = "product", fallback = ProductClient.ProductClientFallback.class)
public interface ProductClient {

    @PostMapping("/product/listForOrder")
    List<ProductInfoOutput> listForOrder(@RequestBody List<String> productIdList);


    @Component
    static class ProductClientFallback implements ProductClient {

        @Override
        public List<ProductInfoOutput> listForOrder(List<String> productIdList) {
            return null;
        }

    }
}
```

## 熔断控制台

Spring Cloud Hystrix提供了一个控制台，用于查看系统熔断情况

首先需要在pom.xml中加入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency> 
```

在启动类中加入**@EnableHystrixDashboard**

```java
@EnableHystrixDashboard
@SpringCloudApplication
public class OrderApplication {

   public static void main(String[] args) {
      SpringApplication.run(OrderApplication.class, args);
   }
    
}
```

启动应用，然后再浏览器中输入http://localhost:8080/hystrix可以看到如下界面

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019021914314849.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

Hystrix Dashboard共支持三种不同的监控方式

- 默认的集群监控：通过URL:http://turbine-hostname:port/turbine.stream开启，实现对默认集群的监控。

- 指定的集群监控：通过URL:http://turbine-hostname:port/turbine.stream?cluster=[clusterName]开启，实现对clusterName集群的监控。

- 单体应用的监控：通过URL:http://hystrix-app:port/hystrix.stream开启，实现对具体某个服务实例的监控。

- Delay：控制服务器上轮询监控信息的延迟时间，默认为2000毫秒，可以通过配置该属性来降低客户端的网络和CPU消耗。

- Title:该参数可以展示合适的标题。

点击 **Monitor Stream**按钮

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190219143211588.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

- **实心圆：**

  1、通过颜色的变化代表了实例的健康程度，健康程度从绿色、黄色、橙色、红色递减。

  2、通过大小表示请求流量发生变化，流量越大该实心圆就越大。所以可以在大量的实例中快速发现故障实例和高压实例。

- **曲线：**
用来记录2分钟内流浪的相对变化，可以通过它来观察流量的上升和下降趋势。

注意：当使用Hystrix Board来监控Spring Cloud Zuul构建的API网关时，Thread Pool信息会一直处于Loading状态。这是由于Zuul默认会使用信号量来实现隔离，只有通过Hystrix配置把隔离机制改成为线程池的方式才能够得以展示。


























































