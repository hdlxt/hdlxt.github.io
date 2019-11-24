---
title: 【Spring Cloud 笔记和总结】六、Spring Cloud Config统一配置中心（Git+Spring Cloud Bus+RabbitMQ+Git WebHook）
date: 2019-11-05 20:30:12
tags:
- Spring Boot
- Spring Cloud 
- 微服务
- 分布式
- 统一配置中心
- Spring Cloud Bus
- RabbitMQ
- Git WebHook
categories:
- Spring
- Spring Cloud 
copyright: true。

---





- ####  一、简介
	基于Spring Cloud Config实现统一配置中心，将配置文件存放于Git(GitHub)上，通过Spring Cloud Bus消息总线&RabbitMQ消息中间件进行服务间消息通信。
	
	**涉及项目**
	
	- exureka-server
	
	- config-server
	
	- config-client
	
	  <!--more-->
	
	**整体架构图大致如下，使用GitHub Webhooks 触发配置中心刷新配置**
	
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117200225699.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	
	上图来源：`http://blog.didispace.com/springcloud7/`
	
	####  二、配置中心服务端实现
	**依赖**	 	
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
	    <artifactId>config-server</artifactId>
	    <version>0.0.1-SNAPSHOT</version>
	    <name>config-server</name>
	    <description>Demo project for Spring Boot</description>
	    <dependencies>
	        <dependency>
	            <groupId>org.springframework.cloud</groupId>
	            <artifactId>spring-cloud-config-server</artifactId>
	        </dependency>
	        <dependency>
	            <groupId>org.springframework.cloud</groupId>
	            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
	        </dependency>
	        <!--集成Git Webhooks之后，使用/monitor即可实现配置更新，通知其他服务-->
	        <dependency>
	            <groupId>org.springframework.cloud</groupId>
	            <artifactId>spring-cloud-config-monitor</artifactId>
	        </dependency>
	        <dependency>
	            <groupId>org.springframework.boot</groupId>
	            <artifactId>spring-boot-starter-actuator</artifactId>
	        </dependency>
	        <dependency>
	            <groupId>org.springframework.cloud</groupId>
	            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
	        </dependency>
	    </dependencies>
	</project>
	
	```
	**配置文件**
	```yml
	server:
	  port: 8080
	spring:
	  application:
	    name: spring-cloud-config-server
	  cloud:
	    config:
	      server:
	        git:
	          uri: https://github.com/hdlxt/springcloud # 配置git仓库的地址
	          search-paths: config-repo                     # git仓库地址下的相对地址，可以配置多个，用,分割。
	          username:                     # git仓库的账号
	          password:           # git仓库的密码
	  rabbitmq:
	    host: 111.231.xxxx.xx
	    port: 5672
	    username: guest
	    password: guest
	eureka:
	  client:
	    serviceUrl:
	      defaultZone: http://localhost:8000/eureka/   #注册中心eurka地址
	management:
	  endpoints:
	    web:
	      exposure:
	       # 2.x手动开启  这个是用来暴露 endpoints 的。由于 endpoints 中会包含很多敏感信息，除了 health 和 info 两个支持 web 访问外，其他的默认不支持 web 访问
	        include: bus-refresh
	
	```
	- 主要内容
		- 添加配置中心文件存放于github
		- 配置rabbitmq连接信息
		- 暴露bus-refresh
	- github配置文件内容	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117211411126.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 仓库中的配置文件会被转换成 Web 接口，访问可以参照以下的规则：
		 - /{application}/{profile}[/{label}]
		- /{application}-{profile}.yml
		- /{label}/{application}-{profile}.yml
		- /{application}-{profile}.properties
		- /{label}/{application}-{profile}.properties
	  - 上面的 URL 会映射 {application}-{profile}.yml 对应的配置文件，其中 {label} 对应 Git 上不同的分支，默认为 master。
		
	
	**启动类比较简单,声明是服务、配置中心服务端**
	```java
	@EnableConfigServer
	@EnableDiscoveryClient
	@SpringBootApplication
	public class ConfigServerApplication {
	
	    public static void main(String[] args) {
	        SpringApplication.run(ConfigServerApplication.class, args);
	    }
	
	}
	```
	到此配置中心服务端完毕。
	
	####  二、配置中心客户端实现
	- 依赖
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
		    <artifactId>config-client</artifactId>
		    <version>0.0.1-SNAPSHOT</version>
		    <name>config-client</name>
		    <description>Demo project for Spring Boot</description>
		    <dependencies>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-config</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.boot</groupId>
		            <artifactId>spring-boot-starter-actuator</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
		        </dependency>
		        <dependency>
		            <groupId>org.springframework.cloud</groupId>
		            <artifactId>spring-cloud-starter-bus-amqp</artifactId>
		        </dependency>
		    </dependencies>
		</project>
		
		```
	- 配置文件
		- 关于` bootstrap.yml`和`application.yml`
			- bootstrap.yml（bootstrap.properties） 用来程序引导时执行，应用于更加早期配置信息读取，如可以使用来配置application.yml中使用到参数等	
			- application.yml（application.properties) 应用程序特有配置信息，可以用来配置后续各个模块中需使用的公共参数等。
			- bootstrap.yml 先于 application.yml 加载
		- bootstrap.yml
			```yml
			server:
			  port: 8003
			spring:
			  cloud:
			    config:
			      name: lxt-config
			      profile: dev
			      label: master
			      discovery:
			        #开启Config服务发现支持
			        enabled: true
			        #指定server端的name,也就是server端spring.application.name的值
			        service-id: spring-cloud-config-server
			eureka:
			  client:
			    service-url:
			      defaultZone: http://localhost:8000/eureka/
			```
	 	- application.yml
			```yml
			spring:
			  application:
			    name: spring-cloud-config-client
			  cloud:
			    bus:
			     trace:
			       # 开启消息跟踪事件
			       enabled: true
			  rabbitmq:
			    host: 111.231.29.249
			    port: 5672
			    username: guest
			    password: guest
			management:
			  endpoints:
			    web:
			      exposure:
			        include: refresh
			
		```
	- 主要代码
		-  核心
			- @EnableDiscoveryClient
			- @RefreshScope:使用该注解的类，会在接到SpringCloud配置中心配置刷新的时候，自动将新的配置更新到该类对应的字段中
		- 启动类
			```bash
			
			@EnableDiscoveryClient
			@SpringBootApplication
			public class ConfigClientApplication {
			
			    public static void main(String[] args) {
			        SpringApplication.run(ConfigClientApplication.class, args);
			    }
			
			}
			```
		- 测试
			```bash
			/**
			 * @author lxt
			 * @Copy Right Information: lxt
			 * @Project: spring cloud
			 * @CreateDate: 2018/12/16 20:04
			 * @history Sr Date Modified By Why & What is modified
			 * 1.2018/12/16 lxt & new
			 */
			@RestController
			@RefreshScope // 使用该注解的类，会在接到SpringCloud配置中心配置刷新的时候，自动将新的配置更新到该类对应的字段中
			public class HelloController {
			    @Value("${lxt.hello}")
			    private String hello;
			
			    @RequestMapping("/hello")
			    public String from() {
			        return this.hello;
			    }
			}
			```
	####  三、整合
	 **以安装包或者docker方式安装RabbitMQ，启动**![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117213514487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	
	 **配置GitHub Webhooks**
	- （可选，云服务器可跳过）内网穿透工具下载[natapp](#https://natapp.cn/)安装和配置（有免费通道），目的可给本机电脑映射一个外网域名，用于配置Webhooks
	![在这里插入图片描述](https://img-blog.csdnimg.cn/2019111721323258.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 配置Webhooks，配置中心服务端需引入`spring-cloud-config-monitor`依赖
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117214212673.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	####  四、测试
	- 分别启动注册中心、配置中心服务端
	- 启动多个客户端
		- 可通过打jar包形式启动
		- 也可通过idea添加启动类，指定不同端口启动
		![在这里插入图片描述](https://img-blog.csdnimg.cn/20191117213813171.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
	- 修改配置文件，push到github上触发回调`http://t23b4j.natappfree.cc/monitor`,通知配置中心服务端更新配置，服务端以mq形式通过消息总线通知客户端更新配置。
	
	####  五、相关
	- 父模块介绍[`传送门`](https://blog.csdn.net/qq_25283709/article/category/9462287)
	- 源码地址[`传送门`](https://github.com/hdlxt/springcloud)
	- 参考
		- http://www.ityouknow.com/spring-cloud.html	 
