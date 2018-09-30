---
layout: post
title: Spring Cloud Config Retry
tags: [spring-cloud, spring, retry]
---

When having a microservices environment we must think of the resiliency of our environment when platform services like `config-service`, `discovery-service` are not available for a short period of time. Let's consider a `customer-service` which uses discovery first bootstrap registering with a `discovery-service` and getting external configuration from the `config-service`. What we want is to be able to start the services in any order.  

We rely on [Spring Retry](https://github.com/spring-projects/spring-retry) which provides the ability to keep re-trying after a failure.

Let's start with the `config-service`. Here we just configured that it registers within 10 seconds with the `discovery-service` instead of the default 30 seconds. 

```yaml
spring:
  application:
    name: config-service

  cloud:
    config:
      server:
        # the server should configure itself from its own remote repository
        bootstrap: true

        git:
          uri: file://${user.dir}/../spring-cloud-services-startup-config

eureka:
  instance:
    lease-renewal-interval-in-seconds: 10

logging.level:
  com.netflix.discovery: trace
```

For the `discovery-service` we include the `spring-retry` dependency 

```xml
    <dependencies>
        ...
        <dependency>
            <groupId>org.springframework.retry</groupId>
            <artifactId>spring-retry</artifactId>
        </dependency>
        ...
    </dependencies>
```

and use the following configuration:

```yaml
spring:
  application:
    name: discovery-service

  cloud:
    config:
      uri: http://localhost:8888
      fail-fast: true
      retry:
        max-attempts: 20

logging.level:
  org.springframework.retry: trace
  com.netflix.discovery: trace
```

This will keep trying maximum 20 times with an initial backoff interval of 1000ms and an exponential multiplier of 1.1 for subsequent backoffs (maximum 2000ms) to connect to the `config-service`.

And last the `customer-service` also need to include the `spring-retry` library and use the following configuration

```yaml
spring:
  application:
    name: customer-service

  cloud.config:
    discovery:
      enabled: true
      service-id: config-service
    fail-fast: true
    retry:
      max-attempts: 60

logging.level:
  org.springframework.retry: trace
```

We make sure that it keeps on re-trying maximum 60 times using the defaults `spring.cloud.config.retry.*` to register with `discovery-service` getting the external configuration from `config-service`. 

The code examples with the external configuration you can find here:

```bash
https://github.com/altfatterz/spring-cloud-services-startup
https://github.com/altfatterz/spring-cloud-services-startup-config
```








