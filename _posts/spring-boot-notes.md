# Spring boot notes

## spring-boot-cache

配置一个缓存, 例如： `cache#1.cachekey1_name_10`

```java
@CacheConfig(cacheNames = {"cache#1"})
```

用`Cacheabe` 的 key 属性指定缓存键的名字, 用SpEL 来将函数签名中的部分组成键名

```java
@Cacheable(key = " 'cachekey1_name' + #param1 ")
public List get(String param1)
```

`@CachePut` 指定可以更新缓存的方法

```java
@CachePut(value = "cache#1", key = " 'cachekey1_name' + #p0.param1 ")
public CacheEntity update(CacheEntity entity)
```

`@EnableCaching` 加到 PayApplication 上面开启 spring-boot-cache 功能



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



## mybatis 和 ORACLE PROCEDURE

更多详细用法查看：

[demo-oracle-mybatis](https://github.com/71mY4ng/demo-oracle-mybatis.git)

oracle 存储过程的声明：

```sql
-- 存储过程签名， 没有返回值， 两个参数
create or replace procedure exec_proc_1 (today in varchar2, nextday in varchar2)

-- 两个参数， 1个返回值
create or replace procedure get_proc_1 (today in varchar2, nextday in varchar2, summary out number)

-- 两个参数， 1个返回值是 sys_cursor 的游标结果集
create or replace procedure get_proc_2 (today in varchar2, nextday in varchar2, list out sys_cursor)
```

__mybatis 对第一种的定义：__

mapper 接口：

```java
public void executeProc1(@Param("today") String today,
                        @Param("nextday") String nextDay);
```

mapper xml 定义：

```xml
<update id="executeProc1" statementType="CALLABLE">
    CALL executeProc1(#{today}, #{nextday})
</update>
```

__mybatis 对第二种的定义：__

mapper 接口方法定义，可见即时有存储过程有返回值也是通过参数来拿，java方法还是定义成返回void

```java
void getProcSummaryOut(@Param("today") String today,
                        @Param("nextday") String lastName,
                        @Param("summary") Map<String, Double> summary);
```

mapper xml 定义， 指定各参数的类型，有如下几种：
* IN
* OUT
* INOUT
具体使用选择看需求

```xml
<update id="getProcSummaryOut" statementType="CALLABLE">
    {CALL exec_proc_out(#{today, jdbcType=VARCHAR, mode=IN},
                        #{nextday, jdbcType=VARCHAR, mode=IN},
                        #{summary.sumpoints, jdbcType=NUMBER, mode=OUT})}
</update>
```


__mybatis 对第三种的定义：__

```xml
<resultMap id="procMapping">
...
</resultMap>

<select id="getProcCursor" resultMap="procMapping">
    {CALL get_proc_2(#{getClientsQuery.departmentId, jdbcType=NUMERIC, mode=IN},
                     #{getClientsQuery.employees, jdbcType=CURSOR, mode=OUT, javaType=java.sql.ResultSet, resultMap=getEmployeesResultSet})}
</select>
```
