---
layout: post
title: Spring Cloud Discovery with Spring Boot Admin
tags: [springbootadmin, springcloud, spring]
---

When registering services in the [Spring Boot Admin](https://github.com/codecentric/spring-boot-admin) dashboard there are two options: 
1. including the spring boot admin client dependency into the service
2. or using Spring Cloud Discovery with a supported implementation (Eureka, Consul, Zookeeper) 
                                        
I prefer using the Spring Cloud Discovery option because it feels more lightweight without including a dependency into the services and most of the time a Spring Cloud Discovery is already used for the services, why not leverage this one as well for Spring Boot Admin.

Here is an example how to set things up [https://github.com/altfatterz/spring-boot-admin-eureka-finchley](https://github.com/altfatterz/spring-boot-admin-eureka-finchley).
The example is using two little services `foo-service` and `bar-service`, the `foo-service` calls the `bar-service` leveraging Eureka based Spring Cloud Discovery.

```java
@SpringBootApplication
@EnableDiscoveryClient 
public class FooServiceApplication {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
        return restTemplateBuilder.build();
    }

    public static void main(String[] args) {
        SpringApplication.run(FooServiceApplication.class, args);
    }
}

@RestController
class FooController {

    private final BarClient barClient;

    public FooController(BarClient barClient) {
        this.barClient = barClient;
    }

    @GetMapping("/")
    public String foo() {
        return "foo";
    }

    @GetMapping("/foobar")
    public String fooBar() {
        return "foo" + barClient.getBar();
    }

}

@Component
class BarClient {

    private final RestTemplate restTemplate;

    public BarClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public String getBar() {
        return restTemplate.getForObject("http://bar-service", String.class);
    }
}
```

The example is using `Spring Cloud Finchley RC1` and `Spring Boot Admin 2.0.0-SNAPSHOT`.  

In the `eureka-server` the following configuration will run `eureka-server` in standalone mode:

```yaml
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
``` 

In the `spring-boot-admin` application by including the following dependency 

```xml
<dependency>
    <groupId>de.codecentric</groupId>
    <artifactId>spring-boot-admin-starter-server</artifactId>
    <version>${spring.boot.admin.version}</version>
</dependency>
```

and using the `@EnableAdminServer` we can have the Spring Boot Admin dashboard up and running.

<p><img src="/images/2018-05-15/spring-boot-admin.png" alt="Spring Boot Admin Dashboard" /></p>

The services registered in Eureka:

<p><img src="/images/2018-05-15/eureka.png" alt="Eureka" /></p>

So far so good, but in almost every case the different services, the `eureka-server` and `spring-boot-admin` is secured by at least a basic authentication. 

Let's first secure the `foo-service`. 

After including the obvious `spring-boot-starter-security` dependency and the followings in the `application.yml`

```yaml
spring:
  security:
    user:
      name: foo
      password: password
      
      
management:
  endpoints:
    web.exposure.include: "*"
  endpoint:
    health:
      show-details: ALWAYS
```       

you still need the followings in order to let the Spring Boot Admin to access the exposed actuator endpoints:

```yaml
eureka:
  instance:
    metadata-map:
      user.name: ${spring.security.user.name}
      user.password: ${spring.security.user.password}
```      

This makes sure that the username and password is sent to Spring Boot Admin during registration:

```json
{
    "registration": {
        "name": "FOO-SERVICE",
        "managementUrl": "http://10.44.66.87:8080/actuator",
        "healthUrl": "http://10.44.66.87:8080/actuator/health",
        "serviceUrl": "http://10.44.66.87:8080/",
        "source": "discovery",
        "metadata": {
            "user.name": "foo",
            "management.port": "8080",
            "jmx.port": "52698",
            "user.password": "******"
        }
    }
}
```

Next, let's secure the `eureka-server`. Again after including the obvious `spring-boot-starter-security` dependency and the following configuration:

```yaml
spring:
  security:
    user:
      name: eureka
      password: password
```

we need to disable CSRF protection which is still an open [issue](https://github.com/spring-cloud/spring-cloud-netflix/issues/2754). Here I configured also to use only HTTP Basic authentication.  

```java
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .httpBasic();
    }
}
```

Next, for all the three services (`foo-service`, `bar-service` and `spring-boot-admin`) which are registered within `eureka-server` you need to override the default
 
```yaml
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
``` 

to the following:

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka:password@localhost:8761/eureka/
```

And at last let's secure the `spring-boot-admin` service. After including again the obvious `spring-security-starter-security` dependency and setting the 

```yaml
spring:
  security:
    user:
      name: admin
      password: password
```  

we need the following `WebSecurityConfiguration`:

```java
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private final String adminContextPath;
    
    public WebSecurityConfig(AdminServerProperties adminServerProperties) {
        this.adminContextPath = adminServerProperties.getContextPath();
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setTargetUrlParameter("redirectTo");

        http.authorizeRequests()
                .antMatchers(adminContextPath + "/assets/**").permitAll()
                .antMatchers(adminContextPath + "/login").permitAll()
                .anyRequest().authenticated()
                .and()
                .formLogin().loginPage(adminContextPath + "/login").successHandler(successHandler).and()
                .logout().logoutUrl(adminContextPath + "/logout").and()
                .httpBasic().and()
                .csrf().disable();
    }
}
```

This way we get a nice looking login form.  

<p><img src="/images/2018-05-15/spring-boot-admin-login.png" alt="Spring Boot Admin Login" /></p>

I have included a third service also `baz-service` where instead of Spring Cloud Discovery the Spring Boot Admin client dependency is used in order to compare the changes regarding configuration. 

There is also [another](https://github.com/altfatterz/spring-boot-admin-eureka-edgware) repository where you can check how was this done using Spring Boot Admin 1.5.7 with services using Spring Cloud Edgware.