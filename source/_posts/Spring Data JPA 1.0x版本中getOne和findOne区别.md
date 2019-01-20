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
 * 返回对具有给定标识符的实体的引用。(延迟加载)
 * @see EntityManager#getReference(Class, Object)
 */
T getOne(ID id);
```

<h2 id='2'>使用说明</h2>

findOne和getOne重点即为翻译部分内容对比，两者加载策略和返回内容不同

- findOne方法为即时加载，执行该方法之后，立即执行查询的sql语句，返回结果，有对应实体则返回实体对象，如果没有实体对象，则返回null。（[Spring Data JPA 2.0X版本之后findOne方法被findById方法替换](http://xiaotong.site/2019/01/13/Spring%20Data%20JPA%202.0X%E7%89%88%E6%9C%AC%E4%B9%8B%E5%90%8EfindOne%E6%96%B9%E6%B3%95%E8%A2%ABfindById%E6%96%B9%E6%B3%95%E6%9B%BF%E6%8D%A2/)）

- getOne方法为延迟加载

  - 执行该方法之后，并不会执行对应的查询sql语句，而是返回一个带有id的代理对象，无论数据库中是否有该主键对应的实体，都不会返回null，即不可用`xxx==null`来判断是否有返回结果。

  - 当获取该代理对象的其他属性时，会执行执行查询sql，如果数据库中有对应的实体，则返回实体对象；如果数据中无实体对象，则抛出`EntityNotFoundException`异常，如下：

    ![exception](/images/20190120/exception.png)


