---
title: 【Spring Cloud 笔记和总结】七、使用Zuul构建微服务网关.md
date: 2019-11-10 20:30:12
tags:
- Spring Boot
- Spring Cloud 
- 微服务
- 分布式
- Zuul
- 网关
categories:
- Spring
- Spring Cloud 
copyright: true。

---



- ####  一、简单微服务网关搭建
	**maven依赖**
	
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
	    <groupId>com.lxt.gateaway</groupId>
	    <artifactId>zuul</artifactId>
	    <version>0.0.1-SNAPSHOT</version>
	    <name>zuul</name>
	    <description>Demo project for Spring Boot</description>
	    <dependencies>
	        <dependency>
	            <groupId>org.springframework.cloud</groupId>
	            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
	        </dependency>
	        <dependency>
	            <groupId>org.springframework.cloud</groupId>
	            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
	        </dependency>
	    </dependencies>
	
	</project>
	
	```
	<!--more-->
	
	**配置文件**
	
	```yml
	spring:
	  application:
	    name: gateway-zuul
	server:
	  port: 8006
	eureka:
	  client:
	    serviceUrl:
	      defaultZone: http://localhost:8000/eureka/   #注册中心eurka地址
	```
	**启动类添加@EnableZuulProxy**
	
	```bash
	package com.lxt.gateaway.zuul;
	
	import org.springframework.boot.SpringApplication;
	import org.springframework.boot.autoconfigure.SpringBootApplication;
	import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
	import org.springframework.context.annotation.Bean;
	
	@SpringBootApplication
	@EnableZuulProxy
	public class ZuulApplication {
	
	    public static void main(String[] args) {
	        SpringApplication.run(ZuulApplication.class, args);
	    }
	}
	
	
	```
	**Spring Cloud Zuul默认配置说明**
	 - 默认情况下，Zuul会代理所有注册到Eureka Server的微服务，并且Zuul的路由规则如下：http://ZUUL_HOST:ZUUL_PORT/微服务在Eureka上的serviceId/**会被转发到serviceId对应的微服务
	 - eg:http://localhost:8006/spring-cloud-consumer/hello/lxt =>http://spring-cloud-consumer/hello/lxt=>http://localhost:9001/hello/lxt
	 - 也可手动配置指定
	 	- 自定义微服务访问路径
			```bash
			zuul:
			  routes:
			    spring-cloud-consumer-hystrix: /test/**
		```
		- 忽略指定微服务
			```bash
			zuul:
			  ignored-services: spring-cloud-consumer-hystrix,spring-cloud-consumer
			```
		- 同时指定微服务ServiceId和对应路径
			```bash
			zuul:
			  routes:
			    config-client:
			      path: /lxt/**
			      serviceId:  spring-cloud-consumer-hystrix
			```
		-  等等...
	
	 
	
	**启动测试**
	- 分别启动exureka-server、service-consumer、service-provider和zuul
	- 浏览器输入`http://localhost:8006/spring-cloud-consumer/hello/lxt`
	- 返回`hello lxt，this is first messge`，测试成功
	
	
	
	####  二、Zuul路由端点
	- 由于 endpoints 中会包含很多敏感信息，除了 health 和 info 两个支持 web 访问外，其他的默认不支持 web 访问，需手动添加配置暴露`routes`路由端点
	
	```yml
	management:
	  endpoints:
	    web:
	      exposure:
	        # 2.x手动开启  这个是用来暴露 endpoints 的。由于 endpoints 中会包含很多敏感信息，除了 health 和 info 两个支持 web 访问外，其他的默认不支持 web 访问
	        include: routes
	```
	- 浏览器`http://localhost:8006/actuator/routes`，返回如下
	
	```bash
	// 20191121213009
	// http://localhost:8006/actuator/routes
	
	{
	  "/spring-cloud-consumer/**": "spring-cloud-consumer",
	  "/spring-cloud-provider/**": "spring-cloud-provider"
	}
	```
	####  三、Zuul中的Filter使用
	**Zuul中默认实现的Filter**
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191121220713248.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- PRE： 这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。
	- ROUTING：这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求，并使用Apache HttpClient或Netfilx Ribbon请求微服务。
	- POST：这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。
	- ERROR：在其他阶段发生错误时执行该过滤器。 除了默认的过滤器类型，Zuul还允许我们创建自定义的过滤器类型。例如，我们可以定制一种STATIC类型的过滤器，直接在Zuul中生成响应，而不将请求转发到后端的微服务。
	
	**自定义Filter示例**
	
	必须包含`token`的参数才可访问，否则直接返回，如下：
	```java
	package com.lxt.gateaway.zuul;
	
	import com.netflix.zuul.ZuulFilter;
	import com.netflix.zuul.context.RequestContext;
	import com.netflix.zuul.exception.ZuulException;
	import org.apache.commons.lang.StringUtils;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	
	import javax.servlet.http.HttpServletRequest;
	
	public class TokenFilter extends ZuulFilter {
	
	    private final Logger logger = LoggerFactory.getLogger(TokenFilter.class);
	
	    @Override
	    public String filterType() {
	        return "pre";
	    }
	
	    @Override
	    public int filterOrder() {
	        return 0;
	    }
	
	    @Override
	    public boolean shouldFilter() {
	        return true;
	    }
	
	    @Override
	    public Object run() throws ZuulException {
	        RequestContext ctx = RequestContext.getCurrentContext();
	        HttpServletRequest request = ctx.getRequest();
	
	        logger.info("--->>> TokenFilter {},{}", request.getMethod(), request.getRequestURL().toString());
	
	        String token = request.getParameter("token");// 获取请求的参数
	
	        if (StringUtils.isNotBlank(token)) {
	            ctx.setSendZuulResponse(true); //对请求进行路由
	            ctx.setResponseStatusCode(200);
	            ctx.set("isSuccess", true);
	            return null;
	        } else {
	            ctx.setSendZuulResponse(false); //不对其进行路由
	            ctx.setResponseStatusCode(400);
	            ctx.setResponseBody("token is empty");
	            ctx.set("isSuccess", false);
	            return null;
	        }
	    }
	}
	
	```
	**注册Bean**
	
	```java
	
	    @Bean
	    public TokenFilter tokenFilter() {
	        return new TokenFilter();
	    }
	```
	**运行测试**
	- 浏览器输入`http://localhost:8006/spring-cloud-consumer/hello/lxt`,返回`token is empty`
	- 浏览器输入`http://localhost:8006/spring-cloud-consumer/hello/lxt?token=2`,返回`hello lxt，this is first messge`
	####  四、Zuul中的路由熔断和重试
	##### 路由熔断
	**实现FallbackProvider接口，重写fallbackResponse方法**
	
	```java
	package com.lxt.gateaway.zuul;
	
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import org.springframework.cloud.netflix.zuul.filters.route.FallbackProvider;
	import org.springframework.http.HttpHeaders;
	import org.springframework.http.HttpStatus;
	import org.springframework.http.MediaType;
	import org.springframework.http.client.ClientHttpResponse;
	
	import java.io.ByteArrayInputStream;
	import java.io.IOException;
	import java.io.InputStream;
	
	public class ProducerFallback implements FallbackProvider{
	    private final Logger logger = LoggerFactory.getLogger(FallbackProvider.class);
	
	    //指定要处理的 service。
	    @Override
	    public String getRoute() {
	        return "spring-cloud-provider";
	    }
	
	    @Override
	    public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
	        if (cause != null && cause.getCause() != null) {
	            String reason = cause.getCause().getMessage();
	            logger.info("Excption {}",reason);
	        }
	        return fallbackResponse();
	    }
	
	    public ClientHttpResponse fallbackResponse() {
	        return new ClientHttpResponse() {
	            @Override
	            public HttpStatus getStatusCode() throws IOException {
	                return HttpStatus.OK;
	            }
	
	            @Override
	            public int getRawStatusCode() throws IOException {
	                return 200;
	            }
	
	            @Override
	            public String getStatusText() throws IOException {
	                return "OK";
	            }
	
	            @Override
	            public void close() {
	
	            }
	
	            @Override
	            public InputStream getBody() throws IOException {
	                return new ByteArrayInputStream("The service is unavailable.".getBytes());
	            }
	
	            @Override
	            public HttpHeaders getHeaders() {
	                HttpHeaders headers = new HttpHeaders();
	                headers.setContentType(MediaType.APPLICATION_JSON);
	                return headers;
	            }
	        };
	    }
	}
	
	```
	**注册bean**
	
	```bash
	    @Bean
	    public ProducerFallback producerFallback() {
	        return new ProducerFallback();
	    }
	```
	**运行测试**
	- 再重启网关`zuul`,启动务提供者`service-provider1`
	- 浏览多次器输入`http://localhost:8006/spring-cloud-provider/foo?foo=lxt&token=2`
	- 交替返回`hello lxt，this is first messge`和`hello lxt，this is two messge`
	- 关闭第二个服务提供者，继续刷新浏览器
	- 交替返回`hello lxt，this is first messge`和`The service is unavailable.`
	##### 路由重试
	**添加依赖**
	
	```bash
	<dependency>
		<groupId>org.springframework.retry</groupId>
		<artifactId>spring-retry</artifactId>
	</dependency>
	```
	
	**修改配置文件**
	
	```bash
	zuul:
	  retryable: true #是否开启重试功能
	ribbon:
	  MaxAutoRetries: 2 #对当前服务的重试次数
	  MaxAutoRetriesNextServer: 0 #切换相同Server的次数
	```
	**修改service-provider1的hello方法如下**
	
	```bash
	    @RequestMapping(value ="/hello", method = RequestMethod.GET)
	    public String index(String name) {
	        logger.info("request two name is "+name);
	        try{
	            Thread.sleep(1000000);
	        }catch ( Exception e){
	            logger.error(" hello two error",e);
	        }
	        return "hello "+name+"，this is two messge";
	    }
	```
	**运行测试**
	- 重启网关`zuul`和`service-provider1`
	- 浏览器输入`http://localhost:8006/spring-cloud-provider/foo?foo=lxt&token=2`
	- 返回`The service is unavailable.`时，查看控制如下：
	
	```bash
	2019-11-21 22:38:31.049  INFO 11240 --- [nio-9002-exec-3] c.l.s.controller.HelloController         : request two name is lxt
	2019-11-21 22:38:32.054  INFO 11240 --- [nio-9002-exec-4] c.l.s.controller.HelloController         : request two name is lxt
	2019-11-21 22:38:33.060  INFO 11240 --- [nio-9002-exec-5] c.l.s.controller.HelloController         : request two name is lxt
	```
	
	- 打印了三次，重试了两次。
	
	
	####  五、相关
	- 父模块介绍[`传送门`](https://blog.csdn.net/qq_25283709/article/category/9462287)
	- 源码地址[`传送门`](https://github.com/hdlxt/springcloud)
	- 参考
		- http://www.ityouknow.com/spring-cloud.html	 
