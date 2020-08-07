# Spring boot cache

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
