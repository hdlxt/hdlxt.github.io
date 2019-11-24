---
title: 【Spring Cloud 笔记和总结】三、服务提供与消费和实现客户端负载均衡
date: 2019-10-18 20:30:12
tags:
- Spring Boot
- Spring Cloud 
- 微服务
- 负载均衡
categories:
- Spring
- Spring Cloud 
copyright: true。

---

#### 一、简介
注册中心Eureka架构图如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191113212752597.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
**分别是注册中心（Eureka）、服务提供（Service Provider）和服务消费（Service Consumer），后两者均为注册到注册中心的服务，因调用关系不同而身份不同，不同的业务场景下身份可能会互换。**

<!--more-->

#### 二、主要内容

###### 1、服务提供者
结构如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191113214038821.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
`HelloController`添加注解`@RestController`,核心代码如下

```java
    @RequestMapping(value = "/hello",method = RequestMethod.GET)
    public String index(String name) {
        return "hello "+name+"，this is first messge";
    }
    @RequestMapping(value = "/foo")
    public String foo(String foo) {
        return "hello "+foo+"，this is first messge";
    }
```
`application.yml`配置文件

```yml
spring:
    application:
        name: spring-cloud-provider
server:
    port: 9000
eureka:
    client:
        serviceUrl:
            defaultZone: http://localhost:8000/eureka/
```
`pom.xml`文件

```bash
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.lxt</groupId>
        <artifactId>springcloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <groupId>com.lxt</groupId>
    <artifactId>service-provider</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>service-provider</name>
    <description>Demo project for Spring Boot</description>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>
</project>

```
###### 2、服务消费者
结构如下，暂时忽略`hystrix`包
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117004121974.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
`HelloRemote`代码

```java
@Component
public class HelloRest {

    @Autowired
    private RestTemplate restTemplate;

    public String hello(String name){
        String response = restTemplate.getForObject("http://spring-cloud-provider/foo?foo="+name,String.class);
        return response;
    }


    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }


}

```

`ConsumerController`核心代码

```bash
@RestController
public class ConsumerController {
    @Autowired
    private HelloRest helloRest;

    @RequestMapping("/hello/{name}")
    public String index(@PathVariable("name") String name) {
        return helloRest.hello(name);
    }
    @RequestMapping("/test")
    public String test() {
        return "test success!";
    }
}
```
`application.yml`配文件

```bash
spring:
    application:
        name: spring-cloud-consumer
server:
    port: 9001
eureka:
    client:
        serviceUrl:
            defaultZone: http://localhost:8000/eureka/
```
`pom.xml`文件

```bash
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.lxt</groupId>
        <artifactId>springcloud</artifactId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <groupId>com.lxt</groupId>
    <artifactId>service-consumer</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>service-consumer</name>
    <description>Demo project for Spring Boot</description>

    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
        </dependency>
    </dependencies>
</project>

```
#### 三、结果演示
- 分别启动注册中心、服务提供者和服务消费者，如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191113220952945.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
- 调用服务消费者测试
		- 浏览器输入`http://localhost:9001/hello/lxt`
		- 返回`hello lxt，this is first messge`，测试成功

#### 四、整合Feign实现负载均衡
**简介**

Fegin是Netfix开发的声明式、模板化的HTTP客户端，Spring Cloud 对Fegin进行了增强，使Fegin支持了Spring MVC注解，并整合了Ribbon和Eureka，从而让Fegin的使用更加方便。

Ribbon是基于Netfix发布的客户端负载均衡器，默认提供了轮询、随机等负载均衡算法，开发者也可以自定义负载均衡算法。

**Eureka Server 和 Fegin整合使用大致架构如下**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117004857162.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
**实现负载均衡**

- 依赖

	```bash
	        <dependency>
	            <groupId>org.springframework.cloud</groupId>
	            <artifactId>spring-cloud-starter-openfeign</artifactId>
	        </dependency>
	```
- 启动类添加注解`@EnableFeignClients`启用feign进行远程调用

	```java
	@SpringBootApplication
	@EnableDiscoveryClient//启用服务注册与发现
	@EnableFeignClients//启用feign进行远程调用
	public class ServiceConsumerApplication {
	
	    public static void main(String[] args) {
	        SpringApplication.run(ServiceConsumerApplication.class, args);
	    }
	
	}
	```

- 使用`@FeignClient`实现负载均衡
	```java
	// name:配置服务提供者名称，用于从注册中心获取服务提供者信息
	@FeignClient(name= "spring-cloud-provider")
	public interface HelloFegin {
	    @RequestMapping(value = "/hello")
	    public String hello(@RequestParam(value = "name") String name);
	
	}
	```
- 复制`service-provider`,重命名`service-provider1`
- 修改配置文件端口为9002
- 修改`HelloController.index`返回值为`"hello "+name+"，this is two messge"`
	如下图：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191113222048474.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
- 注册中心
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191113222346477.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
- 多次测试`http://localhost:9001/hello/lxt`分别返回`hello lxt，this is first messge`和`hello lxt，this is two messge`
#### 五、相关
- 父模块介绍[`传送门`](https://blog.csdn.net/qq_25283709/article/category/9462287)
- 源码地址[`传送门`](https://github.com/hdlxt/springcloud)