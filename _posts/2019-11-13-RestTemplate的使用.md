---
tags: 
- 分布式微服务
---
### 简介

​	在微服务都是以HTTP接口的形式暴露自身服务的，因此在调用远程服务时就必须使用HTTP客户端。我们可以使用JDK原生的`URLConnection`、Apache的`Http Client`、Netty的异步HTTP Client, Spring的`RestTemplate`。但是，用起来最方便、最优雅的还是要属Feign了。这里介绍的是RestTemplate。

spring web 项目提供的RestTemplate，使java访问url更方便，更优雅。

它实现了以下6个主要的HTTP meshod：

| HTTP method |   RestTemplate methods    |
| :---------: | :-----------------------: |
|   DELETE    |          delete           |
|     GET     | getForObject,getForEntity |
|    HEAD     |      headForHeaders       |
|   OPTIONS   |      optionsForAllow      |
|     PUT     |            put            |
|     ANY     |     exchange,execute      |

### Springboot中使用RestTemplate

```java
@RestController
@Slf4j
public class ClientController {

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping("/getProductMsg1")
    public String getProductMsg1() {
        //1.第一种方式(直接使用restTemplate, url写死)
        RestTemplate restTemplate = new RestTemplate();
        String response = restTemplate.getForObject("http://localhost:8080/msg", String.class);
        log.info("response={}", response);
        return response;
    }

    @GetMapping("/getProductMsg2")
    public String getProductMsg2() {

        //2. 第二种方式(利用loadBalancerClient通过应用名获取url, 然后再使用restTemplate)
        RestTemplate restTemplate = new RestTemplate();
        ServiceInstance serviceInstance = loadBalancerClient.choose("PRODUCT");
        String url = String.format("http://%s:%s", serviceInstance.getHost(), serviceInstance.getPort()) + "/msg";
        String response = restTemplate.getForObject(url, String.class);

        log.info("response={}", response);
        return response;
    }

    @GetMapping("/getProductMsg3")
    public String getProductMsg3() {
        //3. 第三种方式(利用配置类加入@LoadBalanced, 可在restTemplate里使用应用名字)
        String response = restTemplate.getForObject("http://PRODUCT/msg", String.class);

        log.info("response={}", response);
        return response;
    }
}
```

配置类

```java
@Component
public class RestTemplateConfig {

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```


