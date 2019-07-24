---
layout: post
title: Conditional caching with Spring @Cacheable annotation 
tags: [spring, caching]
---

In the blog post we look into the `unless` property of the `@Cacheable` Spring annotation when using a custom key generator. 

We have a custom key generator where we generate the cache key from the `Authentication` object stored in the `SecurityContext`

```java
@Component("customKeyGenerator")
@Log4j2
class CustomKeyGenerator implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        String username = SecurityContextHolder.getContext().getAuthentication().getName();
        log.info("CustomKeyGenerator called with '{}'", username);
        return new SimpleKey(username);
    }
}
```

With the `@CacheConfig` annotation we are using the previous custom key generator. 

```java
@Service
@Log4j2
@CacheConfig(keyGenerator = "customKeyGenerator")
class CustomerService {

    @Cacheable(value = "customers", unless = "@monitoring.monitoringUser()")
    public Customer findOne1() {
        log.info("CustomerService was called");
        return new Customer("customer");
    }

}
```

With the `unless` property of the `@Cacheable` we can veto the adding of a customer to the `customers` cache. 
In the example we don't want to cache customers which are used for monitoring. 

The `unless` property accepts a boolean expression which is evaluated after the method has been called. 
With the `@` sign we can reference a method of a bean

```java
@Component(value = "monitoring")
@Log4j2
class Monitoring {

    @Value("${caching.disable.users:#{T(java.util.Collections).emptyList()}}")
    private List<String> users;

    public boolean monitoringUser() {
        String name = SecurityContextHolder.getContext().getAuthentication().getName();

        // do not cache
        if (users.contains(name)) return true;

        return false;
    }
}
```

The running example you can find it here: [https://github.com/altfatterz/conditional-caching-demo/](https://github.com/altfatterz/conditional-caching-demo/)

We configured couple of these users where caching needs to be skipped 

```yaml
caching:
  disable:
    users: bean, hancock
```    

After building the project and starting the application we can access the

```bash
http :8080/customer
```

but we receive a `401 Unauthorized` error. The endpoint needs a JWT token. The included `JwtTokenGenerator` helps to create couple of valid JWT token for testing.

Here is one for the `bean` user:

```bash
export HEADER='Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJiZWFuIiwiYXV0aG9yaXRpZXMiOiJ1c2VyIn0.SM8rLETjXuo8xrrn2OwDb99EcxYHUI7DYZXL271ZdMM'
http :8080/customer $HEADER
```

In the logs we see:

```bash
2019-07-24 22:00:52.962 TRACE 38569 --- [nio-8080-exec-7] o.s.cache.interceptor.CacheInterceptor   : Computed cache key 'SimpleKey [bean]' for operation Builder[public com.example.Customer com.example.CustomerService.findOne()] caches=[customers] | key='' | keyGenerator='customKeyGenerator' | cacheManager='' | cacheResolver='' | condition='' | unless='@monitoring.isMonitoringUser()' | sync='false'
2019-07-24 22:00:52.963 TRACE 38569 --- [nio-8080-exec-7] o.s.cache.interceptor.CacheInterceptor   : No cache entry for key 'SimpleKey [bean]' in cache(s) [customers]
2019-07-24 22:00:52.963  INFO 38569 --- [nio-8080-exec-7] com.example.CustomKeyGenerator           : CustomKeyGenerator called with 'bean'
2019-07-24 22:00:52.963 TRACE 38569 --- [nio-8080-exec-7] o.s.cache.interceptor.CacheInterceptor   : Computed cache key 'SimpleKey [bean]' for operation Builder[public com.example.Customer com.example.CustomerService.findOne()] caches=[customers] | key='' | keyGenerator='customKeyGenerator' | cacheManager='' | cacheResolver='' | condition='' | unless='@monitoring.isMonitoringUser()' | sync='false'
2019-07-24 22:00:52.966  INFO 38569 --- [nio-8080-exec-7] com.example.CustomerService              : CustomerService was called
```

If we call the endpoint again we see that the `CustomerService` is called again, so it is not cached.

```bash
2019-07-24 22:03:02.349  INFO 38569 --- [nio-8080-exec-1] com.example.CustomKeyGenerator           : CustomKeyGenerator called with 'bean'
2019-07-24 22:03:02.349 TRACE 38569 --- [nio-8080-exec-1] o.s.cache.interceptor.CacheInterceptor   : Computed cache key 'SimpleKey [bean]' for operation Builder[public com.example.Customer com.example.CustomerService.findOne()] caches=[customers] | key='' | keyGenerator='customKeyGenerator' | cacheManager='' | cacheResolver='' | condition='' | unless='@monitoring.isMonitoringUser()' | sync='false'
2019-07-24 22:03:02.349 TRACE 38569 --- [nio-8080-exec-1] o.s.cache.interceptor.CacheInterceptor   : No cache entry for key 'SimpleKey [bean]' in cache(s) [customers]
2019-07-24 22:03:02.349  INFO 38569 --- [nio-8080-exec-1] com.example.CustomKeyGenerator           : CustomKeyGenerator called with 'bean'
2019-07-24 22:03:02.349 TRACE 38569 --- [nio-8080-exec-1] o.s.cache.interceptor.CacheInterceptor   : Computed cache key 'SimpleKey [bean]' for operation Builder[public com.example.Customer com.example.CustomerService.findOne()] caches=[customers] | key='' | keyGenerator='customKeyGenerator' | cacheManager='' | cacheResolver='' | condition='' | unless='@monitoring.isMonitoringUser()' | sync='false'
2019-07-24 22:03:02.349  INFO 38569 --- [nio-8080-exec-1] com.example.CustomerService              : CustomerService was called
```

There is also the `condition` property of the `@Cacheble` annotation which works opposite. If the expression is `true` then the method return value is cached.  