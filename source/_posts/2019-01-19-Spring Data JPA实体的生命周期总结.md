---
title: Spring Data JPA实体的生命周期总结
date: 2019-01-19 9:30:12
tags:
- Spring Data JPA
categories:
- Spring
copyright: true
---
Spring Data JPA是对JPA规范的再次封装和抽象，底层使用HIbernate JPA实现，Hibernate实体有三种状态，而Spring Data JPA实体生命周期也有类似的瞬时、托管、删除、游离四种状态，本文记录对实体四种状态的理解和验证过程。
<!--more-->

## 目录

- [**四种状态**](#1)

- [**API示例**](#2)

  - [**persist**](#2.1)
  - [**remove**](#2.2)
  - [**merge**](#2.3)
  - [**refresh**](#2.4)

- [**参考链接**](#3)

  

---

<h2 id='1'>四种状态</h2>

首先以一张图，简单介绍写实体生命周期中四种状态之间的转换关系：

![jpa-entity](/images/20190120/jpa-entity.png)

**瞬时（New）：**瞬时对象，刚New出来的对象，无id，还未和持久化上下文（Persistence Context）建立关联。

**托管（Managed）：**托管对象，有id，已和持久化上下文（Persistence Context）建立关联，对象属性的所有改动均会影响到数据库中对应记录。

 - 瞬时对象调用em.persist（）方法之后，对象由瞬时状态转换为托管状态
 - 通过find、get、query等方法，查询出来的对象为托管状态
 - 游离状态的对象调用em.merge方法，对象由游离状态转换为托管状态

**游离（Datached）：**游离对象，有id值，但没有和持久化上下文（Persistence Context）建立关联。

- 托管状态对象提交事务之后，对象状态由托管状态转换为游离状态
- 托管状态对象调用em.clear()方法之后，对象状态由托管状态转换为游离状态
- New出来的对象，id赋值之后，也为游离状态

**删除（Removed）：**执行删除方法（em.remove()）但未提交事务的对象，有id值，没有和持久化上下文（Persistence Context）建立关联，即将从数据库中删除。

<h2 id='2'>API示例</h2>

> 针对JPA规范的四个方法，写了一个简单的Demo，进行了一一的验证，以下进行验证过程说明，完整代码传送门：https://github.com/hdlxt/SpringDataJpaDemo.git

整体结构如下：

![com.example.demo.controller](/images/20190124/demo.png)

<h3 id='2.1'>persist</h3>

**不同状态下执行em.persist()方法产生结果：**

- 瞬时态：转化为托管态
- 托管态：不发生改变，但执行instert语句
- 删除态：转化为托管态
- 游离态：**抛异常**

**验证删除态和游离态持久化如下**:

**1.持久化删除态**

- 代码

```java
    /**
      * 持久化删除态的对象
      *@param id
      * @return
      */
    @RequestMapping("/persistRemove/{id}")
    public String persistRemove(@PathVariable("id")Long id){
        try {
            User user = userDao.findById(id);
            userDao.persistRemove(user);
        }catch (Exception e){
            logger.error("持久化一个删除态的对象!",e);
            return REPONSE_ERR;
        }
        return REPONSE_SUCCESS;
    }
--------------------------------------------------------
    /**
     * 持久化删除态的对象
     *
     * @param user
     */
    @Override
    public void persistRemove(User user) {
        remove(user);
        persist(user);
        user.setName("persist remove success!");
    }   
```



- 步骤
  - http://localhost:8080/user/persisNew/lxt/001，插入一条数据
  - http://localhost:8080/user/list，检查插入结果，并获取`id`
  - http://localhost:8080/user/persistRemove/{id} ,返回`SUCCESS！`
  - http://localhost:8080/user/list  查看结果
- 结果：结果数据并未删除，而且`name`由`lxt`变为`persist remove success!`

**2.持久化游离态**

- 代码

  ```java
     /**
       * 持久化游离态的对象
       *@param id
       * @return
       */
      @RequestMapping("/persisDetached/{id}")
      public String persisDetached(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.clear();
              userDao.persist(user);
          }catch (Exception e){
              logger.error("持久化一个游离态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  ```

  

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/persisDetached/{id} 返回`ERROR！`

- 结果：抛异常

  ```java
  2019-01-26 00:00:34.090 ERROR 5228 --- [io-8080-exec-10] c.e.demo.controller.UserController       : 持久化一个游离态的对象!
  org.springframework.dao.InvalidDataAccessApiUsageException: detached entity passed to persist: com.example.demo.entity.User; nested exception is org.hibernate.PersistentObjectException: detached entity passed to persist: com.example.demo.entity.User
  	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.convertHibernateAccessException(HibernateJpaDialect.java:317) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:253) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.translateExceptionIfPossible(AbstractEntityManagerFactoryBean.java:527) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.ChainedPersistenceExceptionTranslator.translateExceptionIfPossible(ChainedPersistenceExceptionTranslator.java:61) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.DataAccessUtils.translateIfNecessary(DataAccessUtils.java:242) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	
  	....
  ```

<h3 id='2.2'>remove</h3>

**不同状态下执行em.remove()方法产生结果：**

- 瞬时态：对状态无影响，后台打印日志
- 托管态：转化为托管态
- 删除态：无影响，什么都不发生
- 游离态：抛异常`Removing a detached instance com.example.demo.entity.User...`

**验证过程如下：**

**1.瞬时态**

- 代码

  ```java
      /**
       * 删除new出来的对象
       *@param id
       * @return
       */
      @RequestMapping("/removeNew")
      public String removeNew(){
          try {
              User user = new User().setName("lxt").setNumber("007");
              userDao.remove(user);
          }catch (Exception e){
              logger.error("删除(remove)一个new的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  ```

  

- 步骤

  - http://localhost:8080/user/removeNew

- 结果：返回`SUCCESS!`后台输出日志

  ```java
  2019-01-26 00:17:32.811  INFO 10136 --- [nio-8080-exec-5] o.h.e.i.DefaultDeleteEventListener       : HHH000114: Handling transient entity in delete processing
  ```

  

**2.删除态**

- 代码

  ```java
      /**
       * 删除 删除态对象
       *@param id
       * @return
       */
      @RequestMapping("/removeRemove/{id}")
      public String removeRemove(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.removeRemove(user);
          }catch (Exception e){
              logger.error("删除(remove)一个删除态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  -----------------------------------------------------
     /**
       * 删除 删除态的对象
       *
       * @param user
       */
      @Override
      public void removeRemove(User user) {
          remove(user);
          remove(user);
      }
  ```

  

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/removeRemove/{id} 

- 结果：返回`SUCCESS！`，后台输出一个查询sql和一个删除sql，证明第二个删除没有影响

  ```java
  Hibernate: select user0_.id as id1_0_, user0_.name as name2_0_, user0_.number as number3_0_ from t_user user0_ where user0_.id=?
  Hibernate: delete from t_user where id=?
  ```

  

**3.游离态**

- 代码

  ```java
      /**
       * 删除游离态对象
       *@param id
       * @return
       */
      @RequestMapping("/removeDetached/{id}")
      public String removeDetached(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.removeDetached(user);
          }catch (Exception e){
              logger.error("删除(remove)一个游离态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  -------------------------------------------------------------
     /**
       * 删除游离态的对象
       *
       * @param user
       */
      @Override
      public void removeDetached(User user) {
          clear();
          remove(user);
      }
  ```

  

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/persisDetached/{id} 

- 结果：返回`ERROR!`抛异常

  ```
  2019-01-26 00:14:11.071 ERROR 5228 --- [io-8080-exec-10] c.e.demo.controller.UserController       : 删除(remove)一个游离态的对象!
  
  org.springframework.dao.InvalidDataAccessApiUsageException: Removing a detached instance com.example.demo.entity.User#5; nested exception is java.lang.IllegalArgumentException: Removing a detached instance com.example.demo.entity.User#5
  	at org.springframework.orm.jpa.EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible(EntityManagerFactoryUtils.java:373) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:255) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.translateExceptionIfPossible(AbstractEntityManagerFactoryBean.java:527) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.ChainedPersistenceExceptionTranslator.translateExceptionIfPossible(ChainedPersistenceExceptionTranslator.java:61) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.DataAccessUtils.translateIfNecessary(DataAccessUtils.java:242) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	
  ```

  

<h3 id='2.3'>merge</h3>

**不同状态下执行em.merge()方法产生结果：**

- 瞬时态：提交到数据库，返回一个新的托管态的对象
- 托管态：根据原对象返回一个新的托管态的对象
- 删除态：抛异常`org.springframework.dao.InvalidDataAccessApiUsageException: org.hibernate.ObjectDeletedException: deleted instance passed to merge: [com.example.demo.entity.User#<null>]...`
- 游离态：提交到数据库，进行更新或插入，返回一个新的托管态的对象

**合并（merge）删除态和游离态验证过程如下：**

**1.删除态**

- 代码

  ```java
      /**
       * 持久化删除态的对象
       *@param id
       * @return
       */
      @RequestMapping("/mergeRemove/{id}")
      public String mergeRemove(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.mergeRemove(user);
          }catch (Exception e){
              logger.error("合并(merge)一个删除态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  --------------------------------------------------------------------
      /**
       * 合并删除态的对象
       *
       * @param user
       */
      @Override
      public void mergeRemove(User user) {
          remove(user);
          merge(user);
      }
  ```

  

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/mergeRemove/{id}

- 结果：返回`ERROR`抛异常！

  ```java
  2019-01-26 00:23:01.187  INFO 10136 --- [nio-8080-exec-3] o.h.h.i.QueryTranslatorFactoryInitiator  : HHH000397: Using ASTQueryTranslatorFactory
  Hibernate: select user0_.id as id1_0_, user0_.name as name2_0_, user0_.number as number3_0_ from t_user user0_ where user0_.id=?
  2019-01-26 00:23:01.322 ERROR 10136 --- [nio-8080-exec-3] c.e.demo.controller.UserController       : 合并(merge)一个删除态的对象!
  
  org.springframework.dao.InvalidDataAccessApiUsageException: org.hibernate.ObjectDeletedException: deleted instance passed to merge: [com.example.demo.entity.User#<null>]; nested exception is java.lang.IllegalArgumentException: org.hibernate.ObjectDeletedException: deleted instance passed to merge: [com.example.demo.entity.User#<null>]
  	at org.springframework.orm.jpa.EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible(EntityManagerFactoryUtils.java:373) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:255) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.translateExceptionIfPossible(AbstractEntityManagerFactoryBean.java:527) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.ChainedPersistenceExceptionTranslator.translateExceptionIfPossible(ChainedPersistenceExceptionTranslator.java:61) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	
  ```

**2.游离态**

- 代码

  ```java
      /**
       * 持久化游离态的对象
       *@param id
       * @return
       */
      @RequestMapping("/mergeDetached/{id}")
      public String mergeDetached(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.mergeDetached(user);
          }catch (Exception e){
              logger.error("合并(merge)一个游离态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  ---------------------------------------------------------------
     /**
       * 合并游离态的对象
       *
       * @param user
       */
      @Override
      public void mergeDetached(User user) {
          clear();
          User newUser = merge(user);
          newUser.setName("newUser merge detached success!");
          user.setName("user merge detached success!");
      }
  ```

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/mergeDetached/5，返回`SUCCESS!`
  - http://localhost:8080/user/list，查看

- 结果:对应实体的`name`值变为`newUser merge detached success!`，证明返回新的对象为托管态对象

**2.游离态**

<h3 id='2.4'>refresh</h3>

> 方法可以保证当前的实例与数据库中的实例的内容一致，**注意：是反向同步，将数据库中的数据同步到实体中**

**不同状态下执行em.refresh()方法产生结果：**

- 瞬时态：抛异常`org.springframework.dao.InvalidDataAccessApiUsageException: Entity not managed; `
- 托管态： 将数据库中的数据同步到实体中，返回一个托管态的对象。
- 删除态：抛异常`org.springframework.dao.InvalidDataAccessApiUsageException: Entity not managed; `
- 游离态：抛异常`org.springframework.dao.InvalidDataAccessApiUsageException: Entity not managed; `

**总结：**只有被托管的对象才可以被refresh。

**1.瞬时态**

- 代码

  ```java
      /**
       * 刷新new出来的对象
       *@param id
       * @return
       */
      @RequestMapping("/refreshNew")
      public String refreshNew(){
          try {
              User user = new User().setName("lxt").setNumber("007");
              userDao.refresh(user);
          }catch (Exception e){
              logger.error("刷新(refresh)一个new的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  ```

  

- 步骤

  - http://localhost:8080/user/refreshNew

- 结果：返回`ERROR!`抛异常

  ```java
  2019-01-26 00:38:18.037 ERROR 10136 --- [nio-8080-exec-3] c.e.demo.controller.UserController       : 刷新(refresh)一个new的对象!
  
  org.springframework.dao.InvalidDataAccessApiUsageException: Entity not managed; nested exception is java.lang.IllegalArgumentException: Entity not managed
  	at org.springframework.orm.jpa.EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible(EntityManagerFactoryUtils.java:373) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:255) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.translateExceptionIfPossible(AbstractEntityManagerFactoryBean.java:527) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.ChainedPersistenceExceptionTranslator.translateExceptionIfPossible(ChainedPersistenceExceptionTranslator.java:61) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	
  ```

  

**2.托管态：**

- 代码

  ```java
     /**
       * 刷新托管态对象
       *@param id
       * @return
       */
      @RequestMapping("/refreshManaged/{id}")
      public String refreshManaged(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.refreshManaged(user);
          }catch (Exception e){
              logger.error("刷新(refresh)一个托管态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  ------------------------------------------------------------------
       /**
       * 刷新托管态的对象
       *
       * @param user
       */
      @Override
      public void refreshManaged(User user) {
          user.setName("refresh before!");
          refresh(user);
          logger.info("user:"+user);
      }
  ```

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/refreshManaged/{id},返回`SUCCESS`
  - http://localhost:8080/user/list

- 结果:数据库中数据并无变化，日志打印为数据库中查询出的值，并未打印`refresh before!`

**3.删除态**

- 代码

  ```java
      /**
       * 刷新删除态对象
       *@param id
       * @return
       */
      @RequestMapping("/refreshRemove/{id}")
      public String refreshRemove(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.refreshRemove(user);
          }catch (Exception e){
              logger.error("刷新(refresh)一个删除态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  ----------------------------------------------------------------------   
     /**
       * 刷新删除态的对象
       *
       * @param user
       */
      @Override
      public void refreshRemove(User user) {
         remove(user);
         user.setName("refresh remove before！");
         refresh(user);
          user.setName("refresh remove after！");
      }
  ```

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/refreshRemove/{id}

- 结果：返回`ERROR!`抛异常

  ```java
  2019-01-26 00:40:57.713 ERROR 10136 --- [nio-8080-exec-3] c.e.demo.controller.UserController       : 刷新(refresh)一个删除态的对象!
  
  org.springframework.dao.InvalidDataAccessApiUsageException: Entity not managed; nested exception is java.lang.IllegalArgumentException: Entity not managed
  	at org.springframework.orm.jpa.EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible(EntityManagerFactoryUtils.java:373) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:255) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.translateExceptionIfPossible(AbstractEntityManagerFactoryBean.java:527) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.ChainedPersistenceExceptionTranslator.translateExceptionIfPossible(ChainedPersistenceExceptionTranslator.java:61) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	
  ```

**4.游离态**

- 代码

  ```java
      /**
       * 刷新游离态对象
       *@param id
       * @return
       */
      @RequestMapping("/refreshDetached/{id}")
      public String refreshDetached(@PathVariable("id")Long id){
          try {
              User user = userDao.findById(id);
              userDao.refreshDetached(user);
          }catch (Exception e){
              logger.error("刷新(refresh)一个游离态的对象!",e);
              return REPONSE_ERR;
          }
          return REPONSE_SUCCESS;
      }
  ----------------------------------------------------------------------
     /**
       * 刷新游离态的对象
       *
       * @param user
       */
      @Override
      public void refreshDetached(User user) {
          clear();
          refresh(user);
      }
  ```

- 步骤

  - http://localhost:8080/user/list，获取`id`
  - http://localhost:8080/user/refreshDetached/{id},`

- 结果:返回`ERROR！`抛异常！

  ```java
  2019-01-26 00:42:09.598 ERROR 10136 --- [nio-8080-exec-7] c.e.demo.controller.UserController       : 刷新(refresh)一个游离态的对象!
  
  org.springframework.dao.InvalidDataAccessApiUsageException: Entity not managed; nested exception is java.lang.IllegalArgumentException: Entity not managed
  	at org.springframework.orm.jpa.EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible(EntityManagerFactoryUtils.java:373) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.vendor.HibernateJpaDialect.translateExceptionIfPossible(HibernateJpaDialect.java:255) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.orm.jpa.AbstractEntityManagerFactoryBean.translateExceptionIfPossible(AbstractEntityManagerFactoryBean.java:527) ~[spring-orm-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	at org.springframework.dao.support.ChainedPersistenceExceptionTranslator.translateExceptionIfPossible(ChainedPersistenceExceptionTranslator.java:61) ~[spring-tx-5.1.4.RELEASE.jar:5.1.4.RELEASE]
  	
  ```

  

<h2 id='3'>参考链接</h2>

- [JPA EntityManager的四个主要方法 ——persist,merge,refresh和remove](https://blog.csdn.net/javavenus/article/details/6289616)
- [JPA 实体生命周期理解和总结](https://blog.csdn.net/yingxiake/article/details/50968059)



