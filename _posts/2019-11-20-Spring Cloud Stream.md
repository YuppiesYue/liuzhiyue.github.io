# Spring Cloud Stream

## 简介

​	Spring Cloud Stream 是一个构建消息驱动微服务的框架。应用程序通过 inputs 或者 outputs 来与 Spring Cloud Stream 中binder 交互，通过我们配置来 binding ，而 Spring Cloud Stream 的 binder 负责与消息中间件交互。目前仅支持**RabbitMQ、Kafka**。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190215114106453.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

​	Spring Cloud Stream由一个中间件中立的核组成。应用通过Spring Cloud Stream插入的input(相当于消费者consumer，它是从队列中接收消息的)和output(相当于生产者producer，它是从队列中发送消息的。)通道与外界交流。通道通过指定中间件的Binder实现与外部代理连接。业务开发者不再关注具体消息中间件，只需关注Binder对应用程序提供的抽象概念来使用消息中间件实现业务即可。

​	**Binder**通过定义绑定器作为中间层，实现了应用程序与消息中间件(Middleware)细节之间的隔离。通过向应用程序暴露统一的Channel通过，使得应用程序不需要再考虑各种不同的消息中间件的实现。当需要升级消息中间件，或者是更换其他消息中间件产品时，我们需要做的就是更换对应的Binder绑定器而不需要修改任何应用逻辑 。甚至可以任意的改变中间件的类型而不需要修改一行代码。目前只提供了RabbitMQ和Kafka的Binder实现。Springcloud Stream还有个好处就是像Kafka一样引入了一点分区的概念，像RabbitMQ不支持分区的队列，你用了SpringCloud Stream技术，它就会帮RabbitMQ引入了分区的特性，SpringCloud Stream就是天然支持分区的，我们用起来还是很方便的。

## 使用方式

pom.xml文件加入依赖

```xml
<!--rabbitMQ的支持-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
<!--kafka的支持-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>
```

配置文件

```yaml
spring:
  application:
    name: order
  cloud:
    stream:
      bindings:
        myMessage: #消息名称
          group: order #分组名称
          content-type: application/json #实体对象消息会序列化成json
```

发送客户端配置

```java
import org.springframework.cloud.stream.annotation.Input;
import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.SubscribableChannel;

public interface StreamClient {

    String INPUT = "myMessage";

    @Input(StreamClient.INPUT)
    SubscribableChannel input();

    @Output(StreamClient.INPUT)
    MessageChannel output();

}
```

发送消息的Controller

```java
@RestController
public class SendMessageController {

    @Autowired
    private StreamClient streamClient;

    /**
     * 发送 orderDTO对象
     */
    @GetMapping("/sendMessage")
    public void process() {
        OrderDTO orderDTO = new OrderDTO();
        orderDTO.setOrderId("123456");
        streamClient.output().send(MessageBuilder.withPayload(orderDTO).build());
    }
}
```

消息接收端

```java
import com.imooc.order.dto.OrderDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.stereotype.Component;

@Component
@EnableBinding(StreamClient.class)
@Slf4j
public class StreamReceiver {

    /**
     * 接收orderDTO对象 消息
     * @param message
     */
    @StreamListener(value = StreamClient.INPUT)
    public String process(OrderDTO message) {
        log.info("StreamReceiver: {}", message);
        return "received.";
    }

}
```
















