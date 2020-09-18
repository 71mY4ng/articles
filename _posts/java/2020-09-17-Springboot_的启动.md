---
layout: article
title: 概述 Spring boot 的启动
tags: spring springBoot
---

# Spring boot 的启动


1. `SpringBootAppication.run(args)`

2. `new SpringApplication(primarySource).run(args)`


## 创建 SpringApplication 实例

1. `new SpringApplication(primarySource)`  创建Spring 实例，加载 primarySource

3. `webApplicationType = deduceWebApplicationType()` 推断 web 应用类型，主要是加载的DispatcherServlet 版本不同决定的(reactive, mvc, servlet 的 DispatcherServlet不同)，Spring支持 ReactiveWeb, NONE (非web 项目), ServletWeb 这几种

4. `setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class))` "应用上下文"初始化工具的加载，关系到注册属性资源，激活 profiles等上下文的加载

5. `setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class))` 设置Spring 事件监听器

6. `mainApplicationClass = deduceMainApplicationClass();` 推断主入口应用类; 用 RuntimeException 拿出 stacktrace, 找出main 方法来决定入口类

<!--more-->

__注意__:

*  `getSpringFactoriesInstances(Class<T> type)` 的作用: Spring 资源工厂 

从Spring 应用缓存中获取type 对应的所有实现类路径url, 如果缺少则用 classLoader 获取 `META-INF/spring.factories` 配置文件中指定的全部 Spring 资源。

以上代码获取了 `ApplicationContextInitializer` 和 `ApplicationListener` 接口的实现类路径url，则会把这些实现类url和classLoader一起加载到Spring 应用缓存中，并且立刻实例化这些类。

_ApplicationContextInitializer 在 spring.factories 中的类url_

```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.SharedMetadataReaderFactoryContextInitializer,\
org.springframework.boot.autoconfigure.logging.ConditionEvaluationReportLoggingListener
```

_ApplicationListener 在 spring.factories 中的类url_

```
# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer
```

此方法还会根据SpringBean 上面的 Order注解指定的优先级大小顺序排序，作为加载的顺序。

```java
AnnotationAwareOrderComparator.sort(instances);
```

* `ApplicationContextInitializer` 的作用:

就一个方法

```java
public void initialize(ConfigurableApplicationContext applicationContext);
```

在 prepareContext 期间调用的模板，根据这个接口可以定义很多需要在 Bean 加载之前要做的操作

例如：实现了 ApplicationContextInitializer 的 ServerPortInfoApplicationContextInitializer 是为了注册一个监听内置 WebServer 启动事件的监听器，这样能通用地获取内置 WebServer 实际 listen 的端口，依赖注入 `local.server.port` 的属性


* `ApplicationListener` 的作用: 

```java
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    /**
     * Handle an application event.
     * @param event the event to respond to
     */
    void onApplicationEvent(E event);

}
```

事件订阅和监听，使用JDK的 `java.util.EventListener` 和 `java.util.EventObject` 接口


## SpringApplication#run(args) 启动SpringApplication

1. 应用上下文和启动异常报告

```java
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
```

run 方法的作用就是配置 `ConfigurableApplicationContext` 返回，`SpringBootExceptionReporter` 则是为了在 SpringApplication 启动失败后打印错误诊断结果

2. SpringApplicationRun 事件监听器

```java
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();   // 发布正在启动的事件
```

`getRunListeners(args)` 加载工厂获取 SpringApplicationRunListener 的关系类然后实例化它们

_SpringApplicationRunListener 在 spring.factories 中的类url_

```
# Run Listeners
org.springframework.boot.SpringApplicationRunListener=\
org.springframework.boot.context.event.EventPublishingRunListener
```

`SpringApplicationRunListeners` 包含的事件类型:
* starting
* environmentPrepared
* contextPrepared
* contextLoaded
* started
* running
* failed

3. 默认应用参数

```java
ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
```

4. 配置应用环境

几个方面

__决定 `ConfigurableEnvironment`__

WebApplicationType 这个在 SpringApplication 实例化的时候加载了一次，这里根据 WebAppType 分为了 StandardServlet 和 Standard 两种环境

为什么要为 Servlet 环境单独做一个参数？

__配置环境__

```java
configureEnvironment(ConfigurableEnvironment environment, String[] args)
```

这一步涉及PropertySources 和 Profiles, 也即是拿到所有配置文件，设置 activeProfiles 属性

2.1 版本有 addConversionService 的操作，这相当是获取解析配置文件格式的parser，比如SpringBoot 就支持 yaml 和 properties 的配置文件解析

以上完成了就会发送 environmentPrepared 事件

5. 打印 Banner ，一个 ASCII art

6. 创建 ApplicationContext

还是根据实例化的 webApplicationType 来决定 ApplicationContext Class 的种类：

* Servlet
* ReactiveWeb
* Default (None-web 项目)

7. 加载 SpringBootExceptionReporter 

这个在 prepareContext 之前，所以在配置好context 之前有任何启动错误就没有错误报告？

```java
exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context)
```

_SpringBootExceptionReporter 在 spring.factories 中的类url_

```
# Error Reporters
org.springframework.boot.SpringBootExceptionReporter=\
org.springframework.boot.diagnostics.FailureAnalyzers
```

8. prepareContext 配置应用上下文

* `context.setEnvironment(environment);`
* `postProcessApplicationContext(context);` applicationContext 的后置处理, 主要配置 Bean生成器和 DefaultResourceLoader
* `applyInitializers(context)` 在 SpringApplication 实例化阶段就决定的 ApplicationContextInitializer，此处就按 Order 顺序加载到 ApplicationContext 中

以上完成, 发出 contextPrepared 事件

然后就开始打印启动日志了 logStartupInfo

__loadContext,bean 的加载__

接下来开始加载 bean

* 第一件事情先把 springApplicationArguments 注册了，保存了命令行配置参数
* 然后是打印 banner的处理类
* 加载所有资源, 先是 primarySources, 然后是其他的 sources
    * BeanDefinition 的资源支持: Class, Resource, Package, CharSequence 这几种。详细一点就是Spring Component Class, XML Bean定义, 包路径通配符（提供给 `ClassPathBeanDefinitionScanner` 来扫描包下面所有的 Component, Repository, Service, Controller, ManagedBean 注解的类), 以及 Environment 配置中的 ${...} 占位符

以上完成，发出 contextLoaded 事件

9. refreshContext,IOC资源和后置处理器

接下来开始进行 context Refresh

refreshContext 的过程涉及到完成配置类的解析，BeanFactoryPostProcessor 和 BeanPostProcessor 等后置处理器的注册, web 内置容器的构造以及 i18n 配置初始化等工作。

注意 BeanFactoryPostProcessor 和 BeanPostProcessor 等都受 PriorityOrdered 和 Ordered 控制启动顺序 ( PriorityOrdered > Ordered > none-ordered )

`BeanFactoryPostProcessor` 涉及的是 bean 定义的后置处理，例如处理 PropertyResource, Configuration 类这种作为bean 生成的依据参数的处理

举例:

* PropertyResource 
    - `PropertyResourceConfigurer` 基类，处理properties 配置文件的解析
    - `PropertyOverrideConfigurer` 处理 properties 中 `beanName.property=value` 的 beanName 覆写
    - `PropertyPlaceholderConfigurer` 把占位符 "${...}" 替换成 BeanDefinition 的处理
* ConfigurationClass
    - `ConfigurationClassPostProcessor` Configuration 注解配置类的解析, 是优先级最高的 `BeanDefinitionRegistryPostProcessor`, 内部涉及到了 @PropertySource, @ComponentScan, @Import, @ImportResource, @Bean 等在 Configuration classs 里面出现的注解处理

`BeanPostProcessor` 则是涉及到与 bean instance 交互需要的后置处理器, 提供 bean 创建初始化前后的钩子，比较典型的作用是处理 @Autowired, @Required, @Resource 这些注解参数的依赖管理和注入:

* @Autowired
    * `AutowiredAnnotationBeanPostProcessor`
* @Required
    * `RequiredAnnotationBeanPostProcessor`
* @PreDestroy, @PostConstruct, @Resource
    * `CommonAnnotationBeanPostProcessor`

`initMessageSource` MessageSource 这个是和 Spring i18n 相关的

`initApplicationEventMulticaster`

`onRefresh` 模板方法，在不同的 ApplicationContext 中做的事情不同，比如 ServletWebServer 会启动 webServer 容器, 目前SpringBoot 支持 tomcat, jetty, undertow 三种内置的 Servlet 容器

`registerListeners` 将 ApplicationListener 的实现类的 beanName 注册到事件广播中

`finishBeanFactoryInitialization` beanFactory 中注册但未实例化的，除了懒加载的所有bean。在这之前经历过 invokeBeanFactoryPostProcessors 等从各种 metadata 中解析出来的 beanDefinition，都会在此初始化，之前注册的 BeanPostProcessor 也随着 bean 创建开始工作。

`finishRefresh` refreshContext 完成后的收尾工作，发布最后的 contextRefreshed 事件

用户可以在 SpringApplication 的 afterRefresh 方法中自定义 contextRefreshed 之后的操作

9. SpringApplication 已经启动，事件 started 发出






参考：

* [fangjian0423/springboot-analysis](https://github.com/fangjian0423/springboot-analysis)