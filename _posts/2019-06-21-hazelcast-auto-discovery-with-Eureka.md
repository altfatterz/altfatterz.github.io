---
layout: post
title: Hazelcast auto-discovery with Eureka
tags: [hazelcast, eureka, springboot]
---

Many modern microservice architectures use a service discovery tool, like [Eureka](https://github.com/Netflix/eureka), that enable a client service to make requests to a dynamically changing set of service instances.
Often these client services need a caching solution when the downstream services are not responding fast enough. [Hazelcast](https://hazelcast.org/) is a popular distributed caching solution and with the [Hazelcast Eureka plugin](https://github.com/hazelcast/hazelcast-eureka) is possible to dynamically configure the nodes leveraging Eureka.

In this blog post we are going to go through a simple example how to achieve this.

The easiest way to start up Eureka is with [Spring Cloud CLI](https://cloud.spring.io/spring-cloud-cli/)  

```bash
$ spring cloud eureka
```

The UI for Eureka will be available at http://localhost:8761/

Next we create a simple use-case leveraging the [Spring Cache abstraction](https://docs.spring.io/spring/docs/current/spring-framework-reference/integration.html#cache):

```java
@Component
@CacheConfig(cacheNames = "customers")
@Slf4j
class CustomerService {
    
    @Cacheable
    public Customer findCustomerById(String id) {
        log.info("Loading customer with id '{}' into cache", id);
        return customerRepository.get(id);
    }

    @CacheEvict(key = "#root.args[0]")
    public void updateCustomer(String id, Customer customer) {
        log.info("Removing customer with id '{}' from the cache", id);
        customerRepository.save(id, customer);
    }

}
``` 

The `Cacheable` annotation first looks into the cache and if is found with the given key (in this case `id`) then it doesn't execute the method, provides the value from the cache.
If it is not found, then executes the method and puts the result into the cache.
With the `@CacheEvit` and `key` parameter we are able to evict values based on a particular key. 

We declare in the `application.yml` default configuration that we want to use `hazelcast` as the cache implementation:

```yaml
spring:
  cache:
    type: hazelcast
```

Next we provide the Hazelcast configuration with Eureka discovery:

```java
public class HazelcastConfiguration {

    @Bean
    public Config hazelcastConfig(EurekaClient eurekaClient) {
        EurekaOneDiscoveryStrategyFactory.setEurekaClient(eurekaClient);
        Config config = new Config();
        config.getNetworkConfig().getJoin().getMulticastConfig().setEnabled(false);
        config.getNetworkConfig().getJoin().getEurekaConfig()
                .setEnabled(true)
                .setProperty("self-registration", "true")
                .setProperty("namespace", "hazelcast")
                .setProperty("use-metadata-for-host-and-port", "true");
        return config;
    }
}
```

We will need the following dependencies:

```bash
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-spring</artifactId>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-eureka-one</artifactId>
    <version>1.1.1</version>
</dependency>
```

Next we start up 2 instances of the `customer-service`

```bash
$ java -jar target/customer-service-0.0.1-SNAPSHOT.jar --server.port=9091
$ java -jar target/customer-service-0.0.1-SNAPSHOT.jar --server.port=9092
```
In the Eureka UI we can also see that the `customer-service` instances are registered with Eureka:

<p><img src="/images/2019-06-21/hazelcast-with-eureka.png" alt="hazelcast-with-eureka" /></p>

If we check the service instances registered with Eureka we can observe that few properties have been set in the `metadata` section

```bash
$ http :8761/eureka/apps/customer-service
```

```xml
<instance>
    <instanceId>192.168.87.65:customer-service:9091</instanceId>
    <hostName>192.168.87.65</hostName>
    <app>CUSTOMER-SERVICE</app>
    ...
    <metadata>
      <management.port>9091</management.port>
      <hazelcast.host>192.168.87.65</hazelcast.host>
      <hazelcast.groupName>dev</hazelcast.groupName>
      <hazelcast.port>5701</hazelcast.port>
    </metadata>
    ...
    <homePageUrl>http://192.168.87.65:9091/</homePageUrl>
    <statusPageUrl>http://192.168.87.65:9091/actuator/info</statusPageUrl>
    <healthCheckUrl>http://192.168.87.65:9091/actuator/health</healthCheckUrl>
    ...
  </instance>
```
  
Using these properties `hazelcast.host`, `hazelcast.groupName`, `hazelcast.port` the service instances can form a cluster as we can see in the logs:

```bash
...
Members {size:2, ver:2} [
	Member [192.168.87.65]:5701 - 699f3487-82b6-4de5-b555-31ebc5d7d4d6
	Member [192.168.87.65]:5702 - d6a98cf2-d7b0-4153-a923-0dd57990aa68 this
]
...
```
  
### Monitoring

Is also important to monitor the cache. The [Hazelcast Management Center](https://hazelcast.com/product-features/management-center/) is good solution for this. However is not free, is limited to only two members.

We can easily start it using:

```bash
docker run -p 8080:8080 hazelcast/management-center:3.12.1
```

and it will be available at `http://localhost:8080/hazelcast-mancenter`

Next, in order to use it, we have to configure it in the `Config` object:

```bash
config.getManagementCenterConfig().setEnabled(true);
config.getManagementCenterConfig().setUrl("http://localhost:8080/hazelcast-mancenter/");
```

### Spring auto-configuration

If we have many services which require a distributed cache solution, we can have a simple Spring auto-configuration which provides the above Hazelcast with Eureka auto-discovery, without needing to copy paste the configuration.

In order to achieve this we create a `hazelcast-playground-autoconfigure` module where we add into the `META-INF/spring.factories`

```bash
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.hazelcast.HazelcastConfiguration
```

where the `HazelcastConfiguration` looks like:

```java
@Configuration
@ConditionalOnClass(Hazelcast.class)
@ConditionalOnMissingBean(HazelcastInstance.class)
@ConditionalOnProperty(value = "spring.cache.type", havingValue = "hazelcast", matchIfMissing = true)
public class HazelcastConfiguration {

    private static final Logger log = LoggerFactory.getLogger(HazelcastConfiguration.class);

    @Bean
    @ConditionalOnClass(value = {EurekaClient.class, MapConfig.class})
    public Config hazelcastConfig(EurekaClient eurekaClient, MapConfig mapConfig) {
        log.info("Using MapConfig with: {}", mapConfig);

        EurekaOneDiscoveryStrategyFactory.setEurekaClient(eurekaClient);
        Config config = new Config();

        config.getManagementCenterConfig().setEnabled(true);
        config.getManagementCenterConfig().setUrl("http://localhost:8080/hazelcast-mancenter/");

        config.addMapConfig(mapConfig);

        config.getNetworkConfig().getJoin().getMulticastConfig().setEnabled(false);
        config.getNetworkConfig().getJoin().getEurekaConfig()
                .setEnabled(true)
                .setProperty("self-registration", "true")
                .setProperty("namespace", "hazelcast")
                .setProperty("use-metadata-for-host-and-port", "true");
        return config;
    }
}
```

The auto-configuration will not be triggered if the `Hazelcast` class is not on the classpath and it will be also not triggered if there is already a `HazelcastInstance` instance configured.

Next we create a `hazelcast-playground-starter-cache` module which contains all the needed runtime dependencies.

The `customer-service` will then depend only on the `hazelcast-playground-starter-cache` and provide only a simple cache configuration (specific to the serivce) like the `size` or `eviction` policy:

```java
@Configuration
public class CacheConfiguration {

    @Bean
    public MapConfig mapConfig() {
        MapConfig mapConfig = new MapConfig("default");
        mapConfig.setMaxSizeConfig(new MaxSizeConfig(300, MaxSizeConfig.MaxSizePolicy.PER_NODE));
        mapConfig.setEvictionPolicy(EvictionPolicy.LRU);
        return mapConfig;
    }
}
```

The example code you can find here: [https://github.com/altfatterz/hazelcast-playground](https://github.com/altfatterz/hazelcast-playground) 