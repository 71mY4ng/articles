---
layout: article
title: Spring Framework Transation Notes
tags: Spring Transactional AOP
---

# Spring Framework Transation Notes

## Spring transaction 的优势

Spring Framework 的 Transation 模型是 __"声明式事务模型"__, 这是其一大优势之一

分成两种使用模型:
* 声明式 (declarative)
* 编程式 (programatic)

管理事务，程序员一般有两种选择：
* 全局事务管理
* 本地事务管理

但两种都存在显著的问题，Spring Framework 对于事务的管理很好地解决了上述两种情况的一些局限性。


<!--more-->


### 全局事务

在此之前，Java EE 关于事务的管理是依靠于繁复的 JTA (Java Transation API)接口，并且JTA依赖于JNDI。而后出现了EJB CMT(Container Managed Transation), 这是声明式事务管理的一种，虽然其减少了对JNDI的处理，但是还是需要写许多java 代码来控制事务，并且需要依赖于EJB，EJB后来式微，EJB CMT 也逐渐没有那么吸引人了。

### 本地事务

本地事务是特定于资源的，举个例子：和JDBC连接对象关联的事务。用法非常简单，但是面临着侵入性很强的事务编程，以及在事务资源这里特定性很强，抽象化很低，例如一段用于JDBC连接的事务代码没法在全局JTA事务中兼容运行。

### Spring 持续编程模型

_placeholder_

## Spring Framework Transation Abstraction

Spring 对事务的抽象关键在于事务策略的概念，事务策略在Spring 框架中被定义成一个接口: `org.springframework.transaction.PlatformTransactionManager`

```Java
public interface PlatformTransactionManager {

    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

    void commit(TransactionStatus status) throws TransactionException;

    void rollback(TransactionStatus status) throws TransactionException;
}
```

这是一个SPI (service provider interface)


`TransactionDefinition` 接口指定了事务的几种概念：

* __Propagation__
    - 通常在事务作用域中的代码都将以这种策略被执行
    - 使用场景有事务挂起 （？）
    - 事务传播和EJB CMT 中的事务传播选项相似
* __Isolation__
    - 定义了该事务与其他事务的隔离程度
    - 举例: 此事务是否能够脏读其他事务的修改？
* __Timeout__
    - 指定一个事务失活时间，超过这个时间的事务将会被背后的事务控制逻辑所回滚
* __Read-only status__
    - 当你只需要读，而不做写操作的时候指定此只读事务
    - 只读事务有时很好地优化了程序的性能，例如在Hibernate 中的事务实现

`TransactionStatus` 定义了一组可供事务代码方便地查询事务状态和控制事务的执行的方法。

```Java
public interface TransactionStatus extends SavepointManager {

    boolean isNewTransaction();

    boolean hasSavepoint();

    void setRollbackOnly();

    boolean isRollbackOnly();

    void flush();

    boolean isCompleted();
}
```

但不管你选择哪种方式去管理事务，一个正确的 `PlatformTransactionManager` 实现是极为关键的，你需要知道如何去在不同的环境下(JTA, Hibernate, JDBC 等)实现这个接口。

如下例子你看到如何使用apache dbcp 工具来指定一个 Jdbc 的 `DataSource` bean。

```XML
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="${jdbc.driverClassName}" />
    <property name="url" value="${jdbc.url}" />
    <property name="username" value="${jdbc.username}" />
    <property name="password" value="${jdbc.password}" />
</bean>

```

与其相关的PlatformTransactionManager包含了一个DataSource 的引用，所以需要这样的配置

```XML
<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

如果你使用java EE 的 JTA, 那么就需要容器管理的 DataSource，并且通过JNDI获取, 从下面配置我们可以看到用的是 JtaTransactionManager，是Spring 提供的在JTA 环境下 `PlatformTransactionManager` 实现。

```XML
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:jee="http://www.springframework.org/schema/jee"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/jee
        https://www.springframework.org/schema/jee/spring-jee.xsd">

    <jee:jndi-lookup id="dataSource" jndi-name="jdbc/jpetstore"/>

    <bean id="txManager" class="org.springframework.transaction.jta.JtaTransactionManager" />

    <!-- other <bean/> definitions here -->

</beans>
```

在以上的配置中，尽管jee container-managed 的 JTA 全局事务或是 JDBC 的本地事务配置有所不同，但差异并不会影响到程序的业务逻辑代码。使用数据访问技术和事务管理技术的不同只会体现在Spring 的配置当中，这也是Spring Framework Transaction Abstraction的优越性之一。



## 使用 Transaction 来同步资源

### 高层同步方法

Spring 提供了高层级的同步方法，在内部处理事务资源的创建，复用，销毁，可选的对资源的事务同步，以及映射多种异常。

* 基于模板方法的持久层集成API
* 原生ORM APIs 加上事务方面(transaction-aware)的工厂bean
* 提供管理原生资源工厂的代理对象

总之，你可以选择使用原生的ORM API 或者用模板方法访问JDBC接口，比如说使用`JdbcTemplate`。

### 低层同步方法

Spring Framework 提供了对应多个环境的工具类来使你直接访问事务资源的原生API:

* `DataSourceUtils` (for JDBC)
* `EntityManagerFactoryUtils` (for JPA)
* `SessionFactoryUtils` (for Hibernate)

例如：获取JDBC 连接不需要直接使用 `java.sql.Connection` 的API，而是使用Spring 提供的 `DataSourceUtils`.

```java
Connection conn = DataSourceUtils.getConnection(dataSource);
```

这么做的意义在于，你能够直接使用原生事务资源对象的API，同时保证，你获取的是Spring管理的资源，事务可以被同步，交互过程中出现的异常能受Spring提供的静态API正确地控制和映射。

使用这些辅助类能让用户不依赖于Spring 的事务控制，但如果你使用过Spring的JDBC, JPA, Hibernate的支持特性，你会发现用Spring 的事务抽象比用底层的辅助类更舒服，例如`JdbcTemplate`，框架能够简化你对数据访问技术的使用。

### 其他辅助

`TransactionAwareDataSourceProxy` 是一个`DataSource`对象的代理，将目标数据源包裹上了一层对Spring管理的事务处理接口。但很少有机会能直接使用这个代理，除非你需要使用JDBC DataSource API 的实现，同时使用Spring 托管的事务。在这种情况下，你可以用Spring 提供的高层事务抽象来实现你的事务代码。


## 声明式事务管理

_placeholder_


## `@Transactional` 注解

#### 启用事务用注解管理

注意开启此功能前需要在 `@Configuration` 类上面加上 `@EnableTransactionManagement`。

或是在xml 中进行以下配置，使得被注解声明的类或方法默认使用配置的 transactionManager 。

```xml
<beans>
    <!-- ... -->
    <!-- enable the configuration of transactional behavior based on annotations -->
    <tx:annotation-driven transaction-manager="txManager"/>

    <!-- ... -->
</beans>
```

#### `@Transactional`和 `<tx:annotation-driven>` 之间的关系？

`@Transactional` 是Spring Transaction Management 的前端元编程手段，单独以此是无法实现声明事务托管的，需要有背后`@Transaction`-aware 的基础架构支撑实际的业务逻辑来使用这个标记。`<tx:annotation-driven>`就声明了运行时遇到`@Transactional`注解需要切换到的事务动作。

### 如何确保`@Transactional`声明被遵守了？

* 注意声明方法的可见性：_推荐声明public级别的方法_
 
`@Transactional` 注解支持在类和方法上面声明，但需要注意方法的可见性：

**使用事务管理代理类的时候，你只能将`@Transactional`标签加在可见性为 `public` 的方法上面**。如果声明标记在`protected`, `private` 或是 包内可见的方法，不会报错，但是声明加入事务管理将不起效，如果你希望在非`public`可见性的方法上使用`@Transactional`注解，可以考虑用AspectJ 来解决。

* 注意代理目标方法和类的方式： _推荐声明在实际的类上（而不是在接口上注解）_

首先你需要知道，Spring 实现支撑事务管理的基础架构依赖于Java 的代理技术，如果Spring 无法通过你的配置创建目标方法的类代理对象，目标方法将不会在事务中执行。

`@Transactional` 可以被加在接口和接口方法上，也能加在类和类方法上，但Spring 官方更推荐你使用后者的方式。

> Spring AOP 框架支持 Java的两种代理技术：JDK Dynamic proxy, CGLIB
> 
> * interface-based proxy (JDK Dynamic proxy 为主，CGLIB 作为前者的备选措施)
>     - JDK Dynamic proxy 即目标类实现了接口，代理类也实现了同样的接口
> * class-based proxy (CGLIB 以及 javassit)
>     - 不需要实现接口，直接创建目标类的子类作为代理类

> `<tx:annotation-driven>` 的选项 `proxy-target-class` 决定了Spring AOP 优先使用哪种代理技术注入，默认情况下是 `<tx:annotation-driven proxy-target-class="false">`，即优先使用 interface-based proxy。

_todo_

* 内部调用无法触发事务：_不要在对象初始化代码中加入事务托管，不要通过内部访问`@Transactional` 注解的方法_

如果`@Transactional` 标记的方法被调用的方式不是来自外部的代理对象，事务就无法触发。

并且需要代理对象被完全创建后才能触发事务管理代码，所以如果你的业务代码写在了初始化代码中，例如`@PostContract`注解的方法中，Spring 将无法遵守将这部分代码加入事务管理的意图。



***参考***

---

- [1]https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html "Spring framework documentation data access technology"
