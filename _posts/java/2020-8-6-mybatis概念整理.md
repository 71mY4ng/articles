---
title: Mybatis 概念整理
tags: Mybatis, 面试题, 面试
notebook: IT剪藏
---

# Mybatis 概念整理


## core concepts

core concepts of Mybatis.

* __SqlSession__: Represents a session with a database, providing users with a way to manipulate the database. 表示了与数据库的连接会话，提供给用户操作数据库的功能

* __MappedStatement__: An abstract representation of Sql that represents instructions to be sent to a database for execution. 抽象化sql 操作，即发送给数据库操作的指令概念表示

* __Executor__: Executor specifically used to interact with the database, accepting MappedStatement as a parameter. Executor 获取 MappedStatement 参数，与数据库交互

* __Mapping interface__: Sql that will be executed in the interface is represented by a method, and the specific Sql is written in the mapping file. SQL 操作被映射为interface 的方法，具体的SQL 写在mapping file 里（Mapping XML）

* __Mapping file__: It can be understood as the place where Mybatis wrote Sql. Usually every single form corresponds to a mapping file in which Sql statements are defined as input and output.  Mybatis 中写SQL 的地方，通常每种形式的SQL 声明都对应一个mapping file

## 缓存

### 一级缓存

默认策略为一级缓存

Mybatis 的一级缓存是 SqlSession 级别，创建一个私有的 hashmap 作为容器

同一个 SqlSession 中，执行两次相同的查询sql，第二次会从缓存中拿

同一个 SqlSession，任意增删改操作会导致缓存失效

当一个 SqlSession 结束，缓存也会清除

### 二级缓存

Mybatis 默认不开启二级缓存

Mybatis 二级缓存是 mapper 级别，同一个 mapper, 多个 SqlSession 共用同一个二级缓存区域

与一级缓存相似的策略，在 __同一个mapper__ 中，__不同__ 的 SqlSession 执行两次相同的查询sql，第二次会从缓存中拿结果

同一个 mapper, 不同 SqlSession 执行任何增删改操作会导致二级缓存清除

所以存在频繁修改操作的数据表不适合开启 Mybatis 的二级缓存
