---
title: Spring Data JPA 2.0X版本之后findOne方法被findById方法替换
date: 2019-01-13 17:30:12
tags:
- Spring Data JPA
categories:
- Spring
copyright: true
---
在使用Spring Boot2.0整合Spring Data JPA时，发现继承`JpaRepository`接口之后无findOne方法，经查阅资料之后，发现已被新的API`findById`方法替换，新的API结合了java8的语法`Optional`（一个专门用于避免空指针NPE而开发的类），使用起来更为方便。
<!--more-->
## 目录
* [**API说明**](#1)
* [**使用示例**](#2)
* [**参考链接**](#3)


---
<h2 id='1'>API说明</h2>

新的API接口如下，去掉了findOne方法，添加了返回值为`Optional<T>`的`findById`方法,调用`findById`方法之后，返回`Optional`的实例，调用`Optional`的`get()`方法即可获取到实体。
```java
package org.springframework.data.repository;
@NoRepositoryBean
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    <S extends T> S save(S var1);

    <S extends T> Iterable<S> saveAll(Iterable<S> var1);
 	//新的根据主键获取实体的方法
    Optional<T> findById(ID var1);

    boolean existsById(ID var1);

    Iterable<T> findAll();

    Iterable<T> findAllById(Iterable<ID> var1);

    long count();

    void deleteById(ID var1);

    void delete(T var1);

    void deleteAll(Iterable<? extends T> var1);

    void deleteAll();
}



package java.util;
public final class Optional<T> {
	....

   /**
     * If a value is present in this {@code Optional}, returns the value,
     * otherwise throws {@code NoSuchElementException}.
     * 如果此@code可选中存在值，则返回该值，否则抛出@code nosuchelementexception。
     * @return the non-null value held by this {@code Optional}
     * @throws NoSuchElementException if there is no value present
     *
     * @see Optional#isPresent()
     */
    public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }

	....
}
```

<h2 id='2'>使用示例</h2>

根据上面的`Optional`的`get()`方法API注释说明可知，直接调用`get()`方法可能会跑出异常，以下为简单参考示例：
```java
	//不建议姿势
	try {
	    User user1 = userDao.findById(1L).get();
	}catch (Exception e){
	    //实体不存在，捕获异常
	}
	
	//相对费劲姿势
	Optional<User> user2 = userDao.findById(1L);
	if(user2.isPresent()){
	    //实体存在
	}else{
	    //实体不存在
	}

	//建议姿势，存在返回实体，不存在不抛异常，返回null
	User user3 = userDao.findById(1L).orElse(null);
```

<h2 id='3'>参考链接</h2>

- https://blog.csdn.net/u012211603/article/details/79828277

	
