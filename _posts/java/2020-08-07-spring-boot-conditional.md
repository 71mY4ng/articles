# Spring boot Conditional

## Conditional

spring boot 文档对于 Conditional 注解的使用例子和解释
* [Conditonal Annotations](https://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#boot-features-condition-annotations)

总的来说这个是结合 `@Bean`, `@Component`, `@Configuration`一起使用的注解，主要作用是对资源的加载进行条件控制。可以通过某个配置项，某个Class 是否加载， 某个Bean 是否加载作为判断条件对Spring 资源的初始化进行控制

Spring 提供了很多 ConditonalOnXXX 的注解对特定类型的资源进行监控，此外，也可以通过实现 Spring 的 `Condition` 接口来自定义自己的条件，并且用 `@Conditional` 进行监听

用例：

** 利用配置项对特性是否开启进行控制 **

```java
@Service
@ConditionalOnProperty(value = "configure.features.enableSignUpLimitFeature")
@CacheConfig(cacheNames = {"profiles"})
public class LimiterProfileCacheService
```

__ps:__ 对于在其它Spring bean中聚合的 Conditional bean有可能出现加载失败的情况， 采取的方法是声名Spring 对这个聚合bean 的依赖是非强制的， 然后在后面做好判断， 就能避免加载错误。 例如： 用`@Autowired(required = false)` 指定资源是可选注入的

