---
tags: 
- 分布式微服务
---
# Spring Cloud Zuul

## 写在前面

​	微服务架构，通常少不了服务网关（API Gateway），服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。Spring Cloud Netflix中的Zuul就担任了这样的一个角色，为微服务架构提供了前门保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。

## 服务网关的要素

- 稳定性、高可用

  网关要保证7*24小时提供服务，如果网关宕机，则所有服务都是通过网关调用的，因为所有服务奖处于不可用的状态。

- 性能、并发

  所有外界调用的请求都是请求网关，可想而知网关的压力是很大的。所以网关的性能必须高。

- 安全性

  网关的安全性也必须高，防止外部恶意访问。例如金融行业需要采用通信的加密。

- 扩展性

  各种请求都经过网关，因此网关上大有文章可做。网关是处理非业务功能的绝佳场所，例如，协议管控、防刷、流量管控、日志监控等。

## 其他网关产品

- Nginx + Lua

  性能和高可用是Nginx的一大特点，先天的事件型驱动设计、全异步的网络IO处理机制、极少的进程间切换以及许多优化设计都使得Nginx天生善于处理高并发请求。扩展性上Nginx上也做的很好，它本身被设计成不同功能，不同层次，不同类型且耦合度极低的模块组成，当对某一模块修复bug或升级时，不用在意其他的模块。

- Kong

  本身就是通过Nginx + Lua进行开发的，但是需要付费。

- Tyk

  开源的，轻量级快速可伸缩的API网关，支持配额和速度限制，支持认证数据分析，支持多用户多组织，提供全REST for API。Tyk是go语言开发的。

## 过滤器

 Zuul实际上是路由+过滤器的结合体，Zuul存在四种过滤器API

- 前置（Pre）

  1、限流

  2、鉴权

  3、参数校验

- 后置（Post）

  1、统计

  2、日志

- 路由（Route）

- 错误（Error）

### 请求的生命周期

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190218192317634.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

- 请求首先到达pre前置过滤器，对请求做一些加工，比如参数校验等；

- 接下来到routing过滤器，将请求路由到指定的服务中去；

- 最后到post后置过滤器，这时候已经获得了请求返回的结果，可以对结果数据做处理或者加工；

- 对于error过滤器则是以上三种过滤器发生异常以后做的处理；

- Zuul也支持自定义过滤器；

## 使用方式

pom.xml文件中引入zuul依赖

```xml
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
</dependency>
```

启动类上加入**@EnableZuulProxy**注解

```java
@SpringBootApplication
@EnableZuulProxy
public class ApiGatewayApplication {

   public static void main(String[] args) {
      SpringApplication.run(ApiGatewayApplication.class, args);
   }

}
```

## 自定义路由配置

```yaml
zuul:
  routes:
  	# 将/product/product/list 路由到 /myProduct/product/list
    aaaaaa:	#可以随便写
      path: /myProduct/**
      serviceId: product
      #敏感头是否过滤，配置成空就是不过滤
      sensitiveHeaders:
  #简洁写法，直接写服务名（serviceId）
  #product: /myProduct/**
  #排除某些路由
  ignored-patterns:
    - /**/product/listForOrder
#访问application/routes可以查看所有路由情况，但是需要权限，下面配置则是关闭权限
management:
  security:
    enabled: false
```

## 前置过滤器（Pre）的应用

### 鉴权

例如给所有请求加上token参数

编写过滤器继承ZuulFilter

```java
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;

import javax.servlet.http.HttpServletRequest;

import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.PRE_DECORATION_FILTER_ORDER;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.PRE_TYPE;

@Component
public class TokenFilter extends ZuulFilter {

    /**
     * 指定该Filter的类型
     * ERROR_TYPE = "error";
     * POST_TYPE = "post";
     * PRE_TYPE = "pre";
     * ROUTE_TYPE = "route";
     */
    @Override
    public String filterType() {
        //过滤器类型为pre前置过滤器
        return PRE_TYPE;
    }

    /**
     * 指定该Filter执行的顺序（Filter从小到大执行）
     * DEBUG_FILTER_ORDER = 1;
     * FORM_BODY_WRAPPER_FILTER_ORDER = -1;
     * PRE_DECORATION_FILTER_ORDER = 5;
     * RIBBON_ROUTING_FILTER_ORDER = 10;
     * SEND_ERROR_FILTER_ORDER = 0;
     * SEND_FORWARD_FILTER_ORDER = 500;
     * SEND_RESPONSE_FILTER_ORDER = 1000;
     * SIMPLE_HOST_ROUTING_FILTER_ORDER = 100;
     * SERVLET_30_WRAPPER_FILTER_ORDER = -2;
     * SERVLET_DETECTION_FILTER_ORDER = -3;
     */
    @Override
    public int filterOrder() {
        //官方推荐写到PRE_DECORATION_FILTER_ORDER之前，则值为PRE_DECORATION_FILTER_ORDER-1
        return PRE_DECORATION_FILTER_ORDER - 1;
    }

    /**
     * 指定需要执行该Filter的规则
     * 返回true则执行run()
     * 返回false则不执行run()
     */
    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();
        String requestUrl = request.getRequestURL().toString();

        // 请求URL内不包含login则需要经过该过滤器，即执行run()
        return !requestUrl.contains("login");
    }

    /**
     * 该Filter具体的执行逻辑
     */
    @Override
    public Object run() {
        RequestContext requestContext = RequestContext.getCurrentContext();
        HttpServletRequest request = requestContext.getRequest();

        //这里从url参数里获取, 也可以从cookie, header里获取
        String token = request.getParameter("token");
        if (StringUtils.isEmpty(token)) {
            requestContext.setSendZuulResponse(false);
            requestContext.setResponseStatusCode(HttpStatus.UNAUTHORIZED.value());
        }
        return null;
    }
}
```

### 限流

这里采用比较常见的令牌桶限流的方法，首先会以一定固定的速录向桶中添加令牌，如果桶已经满了，就会被丢弃，外部请求过来的时候，会从桶里拿到令牌，如果拿到令牌，则继续后续业务处理，如果没有拿到令牌则被拒绝。这里应用到Google的guava组件进行编写。

pom.xml需要引入依赖

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>27.0.1-jre</version>
</dependency>
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190218192336331.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

```java
import com.google.common.util.concurrent.RateLimiter;
import com.imooc.apigateway.exception.RateLimitException;
import com.netflix.zuul.ZuulFilter;
import org.springframework.stereotype.Component;

import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.PRE_TYPE;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.SERVLET_DETECTION_FILTER_ORDER;

/**
 * 令牌桶算法限流
 */
@Component
public class RateLimitFilter extends ZuulFilter {

    /**
     * 每秒向令牌桶放多少令牌（谷歌令牌桶算法RateLimiter）
     */
    private static final RateLimiter RATE_LIMITER = RateLimiter.create(100);

    /**
     * 指定该Filter的类型
     * ERROR_TYPE = "error";
     * POST_TYPE = "post";
     * PRE_TYPE = "pre";
     * ROUTE_TYPE = "route";
     */
    @Override
    public String filterType() {
        return PRE_TYPE;
    }

    /**
     * 指定该Filter执行的顺序（Filter从小到大执行）
     * DEBUG_FILTER_ORDER = 1;
     * FORM_BODY_WRAPPER_FILTER_ORDER = -1;
     * PRE_DECORATION_FILTER_ORDER = 5;
     * RIBBON_ROUTING_FILTER_ORDER = 10;
     * SEND_ERROR_FILTER_ORDER = 0;
     * SEND_FORWARD_FILTER_ORDER = 500;
     * SEND_RESPONSE_FILTER_ORDER = 1000;
     * SIMPLE_HOST_ROUTING_FILTER_ORDER = 100;
     * SERVLET_30_WRAPPER_FILTER_ORDER = -2;
     * SERVLET_DETECTION_FILTER_ORDER = -3;
     */
    @Override
    public int filterOrder() {
        //限流的优先级应该是最高的，SERVLET_DETECTION_FILTER_ORDER，则值为SERVLET_DETECTION_FILTER_ORDER-1
        return SERVLET_DETECTION_FILTER_ORDER - 1;
    }

    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        //RATE_LIMITER.tryAcquire()为取令牌操作
        if (!RATE_LIMITER.tryAcquire()) {
            //取不到令牌抛异常
            throw new RateLimitException();
        }
        return null;
    }
}
```

另外针对限流推荐一个GitHub上的程序，支持很多种限流的储存，Redis、Consul、Spring Data JPA、InMemory

https://github.com/marcosbarbero/spring-cloud-zuul-ratelimit



## 后置过滤器（Post）的应用

### 日志

例如给返回结果记录日志

编写过滤器继承ZuulFilter

```java
import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletResponse;

import java.util.UUID;

import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.POST_TYPE;
import static org.springframework.cloud.netflix.zuul.filters.support.FilterConstants.SEND_RESPONSE_FILTER_ORDER;

@Component
@Slf4j
public class addResponseHeaderFilter extends ZuulFilter{

    /**
     * 指定该Filter的类型
     * ERROR_TYPE = "error";
     * POST_TYPE = "post";
     * PRE_TYPE = "pre";
     * ROUTE_TYPE = "route";
     */
    @Override
    public String filterType() {
        //过滤器类型为post前置过滤器
        return POST_TYPE;
    }

    /**
     * 指定该Filter执行的顺序（Filter从小到大执行）
     * DEBUG_FILTER_ORDER = 1;
     * FORM_BODY_WRAPPER_FILTER_ORDER = -1;
     * PRE_DECORATION_FILTER_ORDER = 5;
     * RIBBON_ROUTING_FILTER_ORDER = 10;
     * SEND_ERROR_FILTER_ORDER = 0;
     * SEND_FORWARD_FILTER_ORDER = 500;
     * SEND_RESPONSE_FILTER_ORDER = 1000;
     * SIMPLE_HOST_ROUTING_FILTER_ORDER = 100;
     * SERVLET_30_WRAPPER_FILTER_ORDER = -2;
     * SERVLET_DETECTION_FILTER_ORDER = -3;
     */
    @Override
    public int filterOrder() {
        //SEND_RESPONSE_FILTER_ORDER，SEND_RESPONSE_FILTER_ORDER-1
        return SEND_RESPONSE_FILTER_ORDER - 1;
    }

    /**
     * 指定需要执行该Filter的规则
     * 返回true则执行run()
     * 返回false则不执行run()
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    @Override
    public Object run() {
        log.info("加入日志")
        return null;
    }
}
```

## Zuul处理跨域

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;
import org.springframework.web.filter.CorsFilter;

import java.util.Arrays;

/**
 * 跨域配置
 * C - Cross  O - Origin  R - Resource  S - Sharing
 */
@Configuration
public class CorsConfig {

    @Bean
    public CorsFilter corsFilter() {
        final UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        final CorsConfiguration config = new CorsConfiguration();

        //是否支持cookie跨域
        config.setAllowCredentials(true);
        //原始域
        config.setAllowedOrigins(Arrays.asList("*")); //http:www.a.com
        //允许头信息
        config.setAllowedHeaders(Arrays.asList("*"));
        //允许方法
        config.setAllowedMethods(Arrays.asList("*"));
        //缓存时间，在此时间内，相同的请求则不再进行跨域检查了
        config.setMaxAge(300l);

        //注册
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
}
```








































