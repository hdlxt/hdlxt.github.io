---
title: 【Spring Cloud 笔记和总结】八、使用 Sleuth和Zipkin进行服务跟踪
date: 2019-11-13 20:30:12
tags:
- Spring Boot
- Spring Cloud 
- 微服务
- 分布式
- Sleuth
- Zipkin
- 服务跟踪
categories:
- Spring
- Spring Cloud 
copyright: true。
---

####  一、添加服务跟踪微服务项目zipkin-server
**父pom添加zipkin相关依赖**

<!--more-->

```bash
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.lxt</groupId>
    <artifactId>springcloud</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>pom</packaging>
    <modules>
        <module>eureka-server</module>
        <module>service-provider</module>
        <module>service-provider1</module>
        <module>service-consumer</module>
        <module>service-consumer-hystrix</module>
        <module>hystrix-dashboard-turbine</module>
        <module>service-consumer-node01</module>
        <module>service-consumer-node02</module>
        <module>config-server</module>
        <module>config-server1</module>
        <module>config-client</module>
        <module>zuul</module>
        <module>consul-provider</module>
        <module>consul-consumer</module>
        <module>zipkin-server</module>
    </modules>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <java.version>1.8</java.version>
<!--        <spring-cloud.version>Greenwich.RC1</spring-cloud.version>-->
        <spring-cloud.version>Greenwich.RELEASE</spring-cloud.version>
        <zipkin-version>2.11.8</zipkin-version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>io.zipkin.java</groupId>
                <artifactId>zipkin-server</artifactId>
                <version>${zipkin-version}</version>
            </dependency>
            <dependency>
                <groupId>io.zipkin.java</groupId>
                <artifactId>zipkin-autoconfigure-ui</artifactId>
                <version>${zipkin-version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
        </repository>
    </repositories>

</project>
```
**本身pom文件依赖**

```bash
<dependency>
   <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--zipkin中包含spring-cloud-starter-sleuth,无需再次引入-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-server</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.java</groupId>
    <artifactId>zipkin-autoconfigure-ui</artifactId>
</dependency>
```
**配置文件**

```bash
server:
  port: 9411
spring:
  application:
    name: zipkin-server
eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://localhost:8000/eureka/
management:
  metrics:
    web:
      server:
        auto-time-requests: false

```
**启动类添加@EnableZipkinServer**
```bash
package com.lxt.zipkin;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import zipkin2.server.internal.EnableZipkinServer;

@SpringBootApplication
@EnableZipkinServer
public class ZipkinServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinServerApplication.class, args);
    }

}
```
####  二、改造service-consumer、service-provider和zuul
**分别添加依赖**

```bash
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```
**配置文件分别添加**

```yml
  #重点
  zipkin:
    #        base-url:当你设置sleuth-cli收集信息后通过http传输到zinkin-server时，需要在这里配置
    base-url: http://localhost:9411
    enabled: true
  sleuth:
    sampler:
      #收集追踪信息的比率，如果是0.1则表示只记录10%的追踪数据，如果要全部追踪，设置为1（实际场景不推荐，因为会造成不小的性能消耗）
      probability: 1.0
```
**注意：添加完毕依赖之后，刷新maven。**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191122230730371.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
**运行测试**
- 分别启动注册中心、网关、服务跟踪、服务提供和服务消费者
- 访问服务跟踪界面`http://localhost:9411`
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191122231051856.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
- 浏览器访问`http://localhost:8006/spring-cloud-pr/hello/1?token=2`,查看服务跟踪界面
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019112223112770.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191122231138592.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
####  三、相关
- 父模块介绍[`传送门`](https://blog.csdn.net/qq_25283709/article/category/9462287)
- 源码地址[`传送门`](https://github.com/hdlxt/springcloud)