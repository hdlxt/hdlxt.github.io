---
title: Spring Boot 核心之自动装配实现
date: 2020-06-01 20:30:12
tags:
- Spring Boot
- 自动装备
- 核心
categories:
- Spring
- Spring Boot
copyright: true。
---

####  一、简介和目标
 - 简介：在 Spring Boot 场景下，基于约定大于配置的原则，实现 Spring 组件自动装配的目的。

 - 目标：完成一个可通过配置和@EnableXXX 来控制的是否装配的Bean

   <!--more-->

####  二、底层装配技术简述
- Spring 模式注解装配
- Spring @Enable 模块装配
	- 注解驱动方式 eg:
		```java
		@Retention(RetentionPolicy.RUNTIME)
		@Target(ElementType.TYPE) @Documented
		@Import(DelegatingWebMvcConfiguration.class) 
		public @interface EnableWebMvc {
		
		}
		```
		```java
		@Configuration 
		public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
		 
		 }
		```
	- 接口编程方式 eg:
	
		```java
		@Target(ElementType.TYPE) 
		@Retention(RetentionPolicy.RUNTIME) 
		@Documented 
		@Import(CachingConfigurationSelector.class) 
		public @interface EnableCaching { 
		 
		 }
		```

		```java
		public class CachingConfigurationSelector extends AdviceModeImportSelector<EnableCaching> {
		    public String[] selectImports(AdviceMode adviceMode) {
		        switch(adviceMode) {
		        case PROXY:
		            return this.getProxyImports();
		        case ASPECTJ:
		            return this.getAspectJImports();
		        default:
		            return null;
		        }
		    }
		 }
		```
- Spring 条件装配
	- 配置方式 - `@Profile`，根据不同环境进行装配
	- 编程方式 - `@Conditional`
		```java
		@Target({ElementType.TYPE, ElementType.METHOD})
		@Retention(RetentionPolicy.RUNTIME)
		@Documented
		@Conditional({OnClassCondition.class})
		public @interface ConditionalOnClass {
		    Class<?>[] value() default {};
		
		    String[] name() default {};
		}
		```

- Spring 工厂加载机制
	
	- 实现类： SpringFactoriesLoader
 	- 配置资源： META-INF/spring.factories
####  三、实现
#####  1、激活自动装配   @EnableAutoConfiguration
**项目结构如下**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191124205214381.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1MjgzNzA5,size_16,color_FFFFFF,t_70)
**添加核心Bean=>HelloWorld,只有一个hello方法用于测试输出结果**
```bash
@Slf4j
public class HelloWorld {
    public void hello(){
        log.info("hello world 2019!");
    }
}
```
**添加HelloWorldConfiguration用于注册HelloWorld**

```bash
@Slf4j
public class HelloWorldConfiguration {
    @Bean
    public HelloWorld hello(){
        log.info("Load HelloWorld");
        return new HelloWorld();
    }
}

```
**添加HelloWorldImportSelector实现ImportSelector，通过接口编程方式实现@Enable功能**

```bash
@Slf4j
public class HelloWorldImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        log.info("annotationMetadata.getAnnotationTypes():{}",annotationMetadata.getAnnotationTypes());
        // 此处可写分支条件，根据指定条件选择性注册某些类 或者返回null

        return new String[]{HelloWorldConfiguration.class.getName()};
    }
}
```
**添加@EnableHelloWorld用于控制是否装配HelloWorld**

```bash
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
//@Import(HelloWorldConfiguration.class) // 基于注解驱动实现Sprig @Enable模块
@Import(HelloWorldImportSelector.class) // 基于接口驱动实现Spring @Enable模块
public @interface EnableHelloWorld {
}
```
`注:如果直接使用@Import(HelloWorldConfiguration.class)注解方式实现，则不需要HelloWorldImportSelector类，但是注解方式无法添加分支判断，只能指定加载指定类`
#####  2、实现自动装配配置类  HelloWorldAutoConfiguration
```bash
@Configuration // 模式注解，声明是一个bean
@ConditionalOnSystemProperty(name = "user.name", value = "Administrator") // 正确的条件装配
//@ConditionalOnSystemProperty(name = "user.name", value = "lxt") // 错误的条件装配
@EnableHelloWorld // Spring @Enable 模块装配
public class HelloWorldAutoConfiguration {
}

```
- 先根据条件注解@ConditionalOnSystemProperty判断是否满足
- 满足则执行@EnableHelloWorld，加载HelloWorldImportSelector，注册HelloWorldConfiguration进而注册HelloWorld
#####  3、配置自动装配实现 META-INF/spring.factories
**在resources下添加META-INF/spring.factories配置文件，用于启动是通过工厂机制（SpringFactoriesLoader）加载**
```bash
# 自动装配
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.lxt.springboot.autoconfigure.configuration.HelloWorldAutoConfiguration
```
#####  4、测试
**添加测试类HelloWorldService**

```bash
@Component
public class HelloWorldService {

    @Autowired
    private HelloWorld helloWorld;

    @PostConstruct
    public void init(){
        helloWorld.hello();
    }

}

```
**启动项目，控制台输出如下，测试开启情况成功**

```bash
2019-11-24 21:25:02.299  INFO 13880 --- [           main] c.l.s.a.a.HelloWorldImportSelector       : annotationMetadata.getAnnotationTypes():[com.lxt.springboot.autoconfigure.condition.ConditionalOnSystemProperty, com.lxt.springboot.autoconfigure.annotation.EnableHelloWorld]
2019-11-24 21:25:02.710  INFO 13880 --- [           main] c.l.s.a.c.HelloWorldConfiguration        : Load HelloWorld
2019-11-24 21:25:02.712  INFO 13880 --- [           main] c.l.s.autoconfigure.entity.HelloWorld    : hello world 2019!
```
**去掉@EnableHelloWorld 注解或者条件注解修改为ConditionalOnSystemProperty(name = "user.name", value = "lxt")，分别重启，控制台输出如下，测试关闭情况成功**

```bash
***************************
APPLICATION FAILED TO START
***************************

Description:

Field helloWorld in com.lxt.springboot.autoconfigure.service.HelloWorldService required a bean of type 'com.lxt.springboot.autoconfigure.entity.HelloWorld' that could not be found.

The injection point has the following annotations:
	- @org.springframework.beans.factory.annotation.Autowired(required=true)


Action:

Consider defining a bean of type 'com.lxt.springboot.autoconfigure.entity.HelloWorld' in your configuration.
```

####  五、源码
 - https://github.com/hdlxt/dive-in-spring-boot