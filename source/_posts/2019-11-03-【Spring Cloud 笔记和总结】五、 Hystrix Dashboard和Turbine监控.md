---
title: 【Spring Cloud 笔记和总结】五、 Hystrix Dashboard和Turbine监控
date: 2019-11-03 21:30:12
tags:
- Spring Boot
- Spring Cloud 
- 微服务
- 分布式
- Hystrix Dashboar
- Turbine
- 监控
categories:
- Spring
- Spring Cloud 
copyright: true。

---

- ####  一、Hystrix Dashboard监控
	- 涉及项目
		
		-  service-consumer-hystrix => 基于service-consumer修改
		
	- 依赖
		
		<!--more-->
		
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
		    <artifactId>service-consumer-hystrix</artifactId>
		    <version>0.0.1-SNAPSHOT</version>
		    <name>service-consumer-hystrix</name>
		    <description>Demo project for Spring Boot</description>
		
		    <dependencies>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-openfeign</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.boot</groupId>
		            <artifactId>spring-boot-starter-actuator</artifactId>
		        </dependency>
		    </dependencies>
		
		</project>
		
		
		```
		
	- 配置文件
	
		```yml
		spring:
		  application:
		    name: spring-cloud-consumer-hystrix
		server:
		  port: 9004
		feign:
		  hystrix:
		    enabled: true
		eureka:
		  client:
		    service-url:
		      defaultZone: http://localhost:8000/eureka/
		```
	
	- 启动类添加注解,注意`getServlet()`方法，用于解决spring cloud2 hystrix没有hystrix.stream路径
		```java
		package com.lxt.springcloudconsumerhystrix;
	
		import com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet;
		import org.springframework.boot.SpringApplication;
		import org.springframework.boot.autoconfigure.SpringBootApplication;
		import org.springframework.boot.web.servlet.ServletRegistrationBean;
		import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
		import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
		import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
		import org.springframework.cloud.openfeign.EnableFeignClients;
		import org.springframework.context.annotation.Bean;
		
		@SpringBootApplication
		@EnableDiscoveryClient
		@EnableFeignClients
		@EnableHystrixDashboard
		@EnableCircuitBreaker
		public class SpringCloudConsumerHystrixApplication {
		
		    public static void main(String[] args) {
		        SpringApplication.run(SpringCloudConsumerHystrixApplication.class, args);
		    }
		
		    /**
		     * 解决：找不到/hystrix.stream 报错：=>Unable to connect to Command Metric Stream.
		     * @return
		     */
		    @Bean
		    public ServletRegistrationBean getServlet() {
		        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
		        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
		        registrationBean.setLoadOnStartup(1);
		        registrationBean.addUrlMappings("/hystrix.stream");
		        registrationBean.setName("HystrixMetricsStreamServlet");
		        return registrationBean;
		    }
		}
		```
	
	
		```
	- 启动项目，浏览器输入`http://localhost:9004/hystrix`,如下：
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117123726133.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
		- 监控默认集群：http://turbine-hostname:port/turbine.stream 
		- 监控指定集群：http://turbine-hostname:port/turbine.stream?cluster=[clusterName]
		- 监控的那个应用：http://hystrix-app:port/hystrix.stream
	- 输入`http://localhost:9004/hystrix.stream `进入监控页面，此时页面显示`Loading`
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117124843621.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117125004752.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 浏览器访问`http://localhost:9004/hello/lxt`之后，监控页面如下
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117124832598.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 图中参数解释如下
	界面解读
	![在这里插入图片描述](https://img-blog.csdnimg.cn/2019111712515840.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	
	实心圆：它有颜色和大小之分，分别代表实例的监控程度和流量大小。如上图所示，它的健康度从绿色、黄色、橙色、红色递减。通过该实心圆的展示，我们就可以在大量的实例中快速的发现故障实例和高压力实例。
	
	曲线：用来记录 2 分钟内流量的相对变化，我们可以通过它来观察到流量的上升和下降趋势。
	其他一些数量指标如下图所示
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117125216734.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 测试成功
	####  二、Turbine集群监控
	  - 涉及项目
	  	- hystrix-dashboard-turbine 
	  	- service-consumer-node01  => 基于service-consumer-hystrix
	  	- service-consumer-node02  => 基于service-consumer-hystrix
	  	- 
	 **hystrix-dashboard-turbine项目添加**
	
	
	-  依赖
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
		    <artifactId>hystrix-dashboard-turbine</artifactId>
		    <version>0.0.1-SNAPSHOT</version>
		    <name>hystrix-dashboard-turbine</name>
		    <description>Demo project for Spring Boot</description>
		    <dependencies>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-turbine</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.boot</groupId>
		            <artifactId>spring-boot-starter-actuator</artifactId>
		        </dependency>
		    </dependencies>
		
		</project>
		
		```
	- 配置文件
		```yml
		spring:
		  application:
		    name: turbine-client
		server:
		  port: 9009
		turbine:
		  # 需要监控的应用名称，默认逗号隔开，内部使用Stringutils.commaDelimitedListToStringArray分割
		  app-config: hystrix-client1,hystrix-client2
		  aggregator:
		    cluster-config: default
		  # 集群名称
		  cluster-name-expression: new String("default")
		  combine-host-port: true
		eureka:
		  client:
		    service-url:
		      defaultZone: http://localhost:8000/eureka/
		  instance:
		    # 启用ip配置 这样在注册中心列表中看见的是以ip+端口呈现的
		    prefer-ip-address: true
		    # 实例名称  最后呈现地址：ip:2000
		    instance-id: ${spring.cloud.client.ip-address}:${server.port}
	
		```
	- 启动类
		
		```java
		package com.lxt.hystrixdashboardturbine;
		
		import com.netflix.hystrix.contrib.metrics.eventstream.HystrixMetricsStreamServlet;
		import org.springframework.boot.SpringApplication;
		import org.springframework.boot.autoconfigure.SpringBootApplication;
		import org.springframework.boot.web.servlet.ServletRegistrationBean;
		import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
		import org.springframework.cloud.netflix.turbine.EnableTurbine;
		import org.springframework.context.annotation.Bean;
		
		@SpringBootApplication
		@EnableHystrixDashboard
		@EnableTurbine
		public class HystrixDashboardTurbineApplication {
		
		    public static void main(String[] args) {
		        SpringApplication.run(HystrixDashboardTurbineApplication.class, args);
		    }
		
		    /**
		     * @return
		     */
		    @Bean
		    public ServletRegistrationBean getServlet() {
		        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
		        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
		        registrationBean.setLoadOnStartup(1);
		        registrationBean.addUrlMappings("/actuator/hystrix.stream");
		        registrationBean.setName("HystrixMetricsStreamServlet");
		        return registrationBean;
		    }
		}
		```
	**service-consumer-node01/02的依赖、代码和service-consumer-hystrix一样，区别在于一个配置，如下**
	- 配置文件,关键点`management.endpoints.web.exposure.include：hystrix.stream`
		```yml
		spring:
		  application:
		    name: hystrix-client1
		server:
		  port: 9005
		
		feign:
		  hystrix:
		    enabled: true
		management:
		  endpoints:
		    web:
		      exposure:
		        # 2.x手动开启  这个是用来暴露 endpoints 的。由于 endpoints 中会包含很多敏感信息，除了 health 和 info 两个支持 web 访问外，其他的默认不支持 web 访问
		        include: hystrix.stream
		eureka:
		  client:
		    service-url:
		      defaultZone: http://localhost:8000/eureka/
		
		```
	**测试**
	- 分别启动注册中心、服务提供者、Turbine监控和两个服务消费者(service-consumer-node01/02)
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117143021668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	**TURBINE-CLIENT再注册中心显示ip:port**
	- 浏览器访问`http://localhost:9009/hystrix`,监控默认集群
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117143330676.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117143406677.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 浏览器访问`http://localhost:9005/hello/lxt`和`http://localhost:9006/hello/lxt`
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117143526476.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 测试成功
	####  三、相关
	- 父模块介绍[`传送门`](https://blog.csdn.net/qq_25283709/article/category/9462287)
	- 源码地址[`传送门`](https://github.com/hdlxt/springcloud)
	- 参考
		- https://www.cnblogs.com/carrychan/p/9529418.html
		- http://www.ityouknow.com/spring-cloud.html	
