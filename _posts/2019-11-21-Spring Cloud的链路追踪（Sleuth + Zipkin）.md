# Spring Cloud的链路追踪（Sleuth + Zipkin）

​	微服务架构中，系统间调用往往会出现延迟与异常等情况，因此，链路追踪工具已经是必不可少的组件，Spring Cloud中集成了这样的组件，那就是Sleuth + Zipkin。

## Spring Cloud Sleuth + Zipkin

### 使用方式

pom.xml文件中引入依赖

```xml
<!--包含sleuth和zipkin-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

### 基本术语

Spring Cloud Sleuth采用的是Google的开源项目Dapper的专业术语。

- **Span**：基本工作单元，发送一个远程调度任务 就会产生一个Span，Span是一个64位ID唯一标识的，Trace是用另一个64位ID唯一标识的，Span还有其他数据信息，比如摘要、时间戳事件、Span的ID、以及进度ID。
- **Trace**：一系列Span组成的一个树状结构。请求一个微服务系统的API接口，这个API接口，需要调用多个微服务，调用每个微服务都会产生一个新的Span，所有由这个请求产生的Span组成了这个Trace。
- **Annotation**：用来及时记录一个事件的，一些核心注解用来定义一个请求的开始和结束 。这些注解包括以下：
  - cs - Client Sent -客户端发送一个请求，这个注解描述了这个Span的开始
  - sr - Server Received -服务端获得请求并准备开始处理它，如果将其sr减去cs时间戳便可得到网络传输的时间。
  - ss - Server Sent （服务端发送响应）–该注解表明请求处理的完成(当请求返回客户端)，如果ss的时间戳减去sr时间戳，就可以得到服务器请求的时间。
  - cr - Client Received （客户端接收响应）-此时Span的结束，如果cr的时间戳减去cs时间戳便可以得到整个请求所消耗的时间。

## Zipkin的安装

官网：https://zipkin.io/

组织架构图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190221142813728.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

### Docker

```dockerfile
docker run -d -p 9411:9411 openzipkin/zipkin
```

### Java

```
curl -sSL https://zipkin.io/quickstart.sh | bash -s
java -jar zipkin.jar
```

### Running from Source

```shell
# get the latest source
git clone https://github.com/openzipkin/zipkin
cd zipkin
# Build the server and also make its dependencies
./mvnw -DskipTests --also-make -pl zipkin-server clean install
# Run the server
java -jar ./zipkin-server/target/zipkin-server-*exec.jar
```

安装完成后

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190221142822293.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE0MjgyNzQ=,size_16,color_FFFFFF,t_70)

配置文件中加入配置，将数据导入Zipkin

```yaml
spring:
  application:
    name: order
  zipkin:
    base-url: http://zipkin:9411/
    sender:
      type: web
  sleuth:
    sampler:
      #抽样百分比，默认10%的数据发到zipkin，1为100%
      probability: 1
```








































































