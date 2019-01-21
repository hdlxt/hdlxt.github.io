---
title: Spring Data JPA实体的生命周期总结
date: 2019-01-19 9:30:12
tags:
- Spring Data JPA
categories:
- Spring
copyright: true
---
Spring Data JPA是对JPA规范的再次封装和抽象，底层使用HIbernate JPA实现，Hibernate实体有三种状态，而Spring Data JPA实体生命周期也有类似的瞬时、托管、删除、游离四种状态，本文记录对实体四种状态的理解和示例。
<!--more-->

## 目录

- [**四种状态**](#1)

- [**API示例**](#2)

  - [**persist**](#2.1)
  - [**remove**](#2.2)
  - [**merge**](#2.3)
  - [**refresh**](#2.4)

  

---

<h2 id='1'>四种状态</h2>

首先以一张图，简单介绍写实体生命周期中四种状态之间的转换关系：

![jpa-entity](/images/20190120/jpa-entity.png)

**瞬时（New）：**瞬时对象，刚New出来的对象，无id，还未和持久化上下文（Persistence Context）建立关联。

**托管（Managed）：**托管对象，有id，已和持久化上下文（Persistence Context）建立关联，对象属性的所有改动均会影响到数据库中对应记录。

 - 瞬时对象调用em.persist（）方法之后，对象由瞬时状态转换为托管状态
 - 通过find、get、query等方法，查询出来的对象为托管状态
 - 游离状态的对象调用em.merge方法，对象由游离状态转换未托管状态

**游离（Datached）：**游离对象，有id值，但没有和持久化上下文（Persistence Context）建立关联。

- 托管状态对象提交事务之后，对象状态由托管状态转换为游离状态
- 托管状态对象调用em.clear()方法之后，对象状态由托管状态转换为游离状态

**删除（Removed）：**删除的对象，有id值，但没有和持久化上下文（Persistence Context）建立关联，即将从数据库中删除。

<h2 id='2'>API示例</h2>

<h3 id='2.1'>persist</h3>

**em.persit():**

<h3 id='2.2'>merge</h3>



<h3 id='2.3'>refresh</h3>





