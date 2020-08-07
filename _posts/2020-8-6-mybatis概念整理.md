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

## 