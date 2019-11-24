---
title: 【Spring Cloud 笔记和总结】四、熔断器Hystrix简单实现
date: 2019-10-25 22:30:12
tags:
- Spring Boot
- Spring Cloud 
- 微服务
- 分布式
- Hystrix
- 熔断器
categories:
- Spring
- Spring Cloud 
copyright: true。

---



- ####  一、简单实现
	- 基于上文服务消费者（`service-consumer`）代码。
	
	- 修改配置文件，添加：feign.hystrix.enable:true如下：
	
		<!--more-->
		
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
		feign:
		  hystrix:
		    enabled: true
		```
		
	- 添加容错回调类
		
		```java
		package com.lxt.serviceconsumer.hystrix;
		
		import com.lxt.serviceconsumer.dao.fegin.HelloFegin;
		import org.springframework.stereotype.Component;
		import org.springframework.web.bind.annotation.RequestParam;
		
		/**
		 * @author lxt
		 * @Copy Right Information: lxt
		 * @Project: spring cloud
		 * @CreateDate: 2018/12/16 15:58
		 * @history Sr Date Modified By Why & What is modified
		 * 1.2018/12/16 lxt & new
		 */
		@Component
		public class HelloFeginHystrix implements HelloFegin {
		    @Override
		    public String hello(@RequestParam(value = "name") String name) {
		        return "hello " +name+", this messge send failed ";
		    }
		}
		
		```
		
	- 修改`HelloFegin`添加回调`fallback`属性
	
		```bash
		@FeignClient(name= "spring-cloud-provider",fallback = HelloFeginHystrix.class)
		```
		
	- 测试
		- 分别启动注册中心、服务提供者和服务消费者 	
		- 浏览器访问`localhost:9001/hello/lxt`,返回`hello lxt，this is first messge`
		- 停止服务提供者，再次访问
		- 返回`hello lxt，this messge send failed`
		- 测试成功
	####  二、相关
	- 父模块介绍[`传送门`](https://blog.csdn.net/qq_25283709/article/category/9462287)
	- 源码地址[`传送门`](https://github.com/hdlxt/springcloud)
	- 参考
		- https://www.cnblogs.com/carrychan/p/9529418.html
