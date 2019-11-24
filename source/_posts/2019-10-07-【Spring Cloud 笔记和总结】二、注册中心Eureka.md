---
title: 【Spring Cloud 笔记和总结】二、注册中心Eureka
date: 2019-10-07 21:05:12
tags:
- Spring Boot
- Spring Cloud 
- 微服务
- 分布式
- 注册中心
- Eureka
categories:
- Spring
- Spring Cloud 
copyright: true。

---

#### 一、关于注册中心
**主要功能如下**

- 服务注册表：记录分布式架构中所有服务和服务地址的映射关系，用于服务直接相互调用	

- 服务注册与发现：服务启动时将自己的信息注册到注册中心；服务直接相互调用时从注册中心获取目标服务信息

- 服务健康检查 ：使用一定机制检查注册中心的服务是否正常，如果长时间无法访问，则将其移除

  <!--more-->

**常见注册中心**
- Eureka

- Consul

- Zookeeper

- Nacos
  ...
  `本文以Eureka为例，后续会更新其他注册中心`

  <!--more-->

#### 二、主要内容
结构如下
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191112220323391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
配置文件`application.yml`如下

```yml
spring:
  application:
    name: spring-cloud-eureka #服务名称
server:
  port: 8000
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false #表示是否将自己注册到Eureka Server，默认为true。
    fetch-registry: false #表示是否从Eureka Server获取注册信息，默认为true。
    serviceUrl:
      defaultZone: http://localhost:8000/eureka #服务地址，多个可用逗号【,】分隔
```
`pom`文件如下

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
    <artifactId>exureka-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>exureka-server</name>
    <description>Demo project for Spring Boot</description>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>
</project>

```
运行结果如下图所示，注册中心目前无服务注册
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191112221221519.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)

#### 三、集群高可用
`C:\Windows\System32\drivers\etc\host`文件添加映射

```bash
127.0.0.1       center
127.0.0.1       center1
127.0.0.1       center2
```

使用yml配置文件连接符`---`添加三个集群的节点`center、center1、center2`,修改后配置文件如下：

```yml


spring:
  application:
    name: spring-cloud-eureka #服务名称
---
server:
  port: 8000
eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: false #表示是否将自己注册到Eureka Server，默认为true。
    fetch-registry: false #表示是否从Eureka Server获取注册信息，默认为true。
    serviceUrl:
      defaultZone: http://localhost:8000/eureka/ #服务地址，多个可用逗号【,】分隔
---
spring:
  profiles: center
server:
  port: 8000
eureka:
  instance:
    hostname: center
  client:
    #register-with-eureka: false #表示是否将自己注册到Eureka Server，默认为true。
    ##fetch-registry: false #表示是否从Eureka Server获取注册信息，默认为true。
    serviceUrl:
      defaultZone: http://center1:8100/eureka/,http://center2:8200/eureka/ #服务地址，多个可用逗号【,】分隔
---
spring:
  profiles: center1
server:
  port: 8100
eureka:
  instance:
    hostname: center1
  client:
    serviceUrl:
      defaultZone: http://center:8000/eureka/,http://center2:8200/eureka/ #服务地址，多个可用逗号【,】分隔
---
spring:
  profiles: center2
server:
  port: 8200
eureka:
  instance:
    hostname: center2
  client:
    serviceUrl:
      defaultZone: http://center:8000/eureka/,http://center1:8100/eureka/ #服务地址，多个可用逗号【,】分隔
---




```
启动：
- mvn package 打成jar包
- cmd 切入到jar所在目录
- 分别运行：

	```bash
	java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=center
	java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=center1
	java -jar spring-cloud-eureka-0.0.1-SNAPSHOT.jar --spring.profiles.active=center2
	```
- 浏览器查看，集群搭建成功
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191112230009761.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
#### 四、相关
- 父模块介绍[`传送门`](https://blog.csdn.net/qq_25283709/article/category/9462287)
- 源码地址[`传送门`](https://github.com/hdlxt/springcloud)