---
title: Spring Data JPA 1.0x版本中getOne和findOne区别
date: 2019-01-13 11:30:12
tags:
- Spring Data JPA
categories:
- Spring
copyright: true
---
上个项目初期，项目成员不熟悉Spring Data JPA框架，在根据主键获取实体时，findOne和getOne混用，留下了不少坑，做个记录，简要说明下区别。
<!--more-->
## 目录
* [**API说明**](#1)
* [**使用说明**](#2)

---
<h2 id='1'>API说明</h2>

```java
/**
 * Retrieves an entity by its id.
 * 
 * @param id must not be {@literal null}.
 * @return the entity with the given id or {@literal null} if none found
 * 返回具有给定ID的实体，如果找不到，则返回@literal null
 * @throws IllegalArgumentException if {@code id} is {@literal null}
 */
T findOne(ID id);
/**
 * Returns a reference to the entity with the given identifier.
 * 
 * @param id must not be {@literal null}.
 * @return a reference to the entity with the given identifier.
 * 返回对具有给定标识符的实体的引用。
 * @see EntityManager#getReference(Class, Object)
 */
T getOne(ID id);
```

findOne和getOne重点即为翻译部分内容对比，两者返回内容不同
- 如果根据id可查询到实体，则findOne返回实体，而getOne返回的是实体的引用，一个代理对象。
- 如果根据id可查询到不实体，则findOne返回null，而getOne返回null的引用，但是null是没有引用的，所以返回了一个异常，如：`Method threw javax.persistence.EntityNotFoundException' exception. Cannot evaluate XXXXX`。

<h2 id='2'>使用说明</h2>

**findOne：**相对于getOne方法比较通用，作为根据主键查询的普通方法即可，根据返回结果是否为`null`来判断是否查询到实体，然后进行下一步操作。  （[Spring Data JPA 2.0X版本之后findOne方法被findById方法替换](http://xiaotong.site/2019/01/13/Spring%20Data%20JPA%202.0X%E7%89%88%E6%9C%AC%E4%B9%8B%E5%90%8EfindOne%E6%96%B9%E6%B3%95%E8%A2%ABfindById%E6%96%B9%E6%B3%95%E6%9B%BF%E6%8D%A2/)）
**getOne：**使用该方法时，当查询不到实体时，返回的是一个异常，但是进行`xxx == null `判断时结果true，注意不可以此作为判断实体是否存在的依据。

	
