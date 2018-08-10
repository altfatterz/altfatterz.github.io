---
layout: post
title: Spring Cloud Services - Service Registry
tags: [springcloud, cloudfoundry, NetflixOSS]
---

Service registry is key component in a microservice architecture which allows applications to dynamically discover and call registered services instead of hand-configuring the used services. 
In this blog post we are going to look into Eureka and into Service Registry (which is based on Eureka) from Spring Cloud Services. Eureka comes from Netflix and has two components Eureka Server and Eureka Client.
The Eureka Server can be embedded in a Spring Boot application using the `@EnableEurekaServer` annotation, but here we look into how to run it using the [Spring Boot CLI](https://cloud.spring.io/spring-cloud-cli/) with [Spring Cloud CLI](https://cloud.spring.io/spring-cloud-cli/) extension installed.  
The `spring cloud --list` lists the available services which can be started

```bash
spring cloud --list
configserver dataflow eureka h2 hystrixdashboard kafka stubrunner zipkin
``` 

By running

```bash
spring cloud eureka
```

Eureka will be available at `http://localhost:8761` 

We have `customser-service` and `order-service` which are two `Eureka Client`s where the `customer-service` calls the `order-service`.

In order that these services register with `Eureka Server` we need to include the following dependency: 

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

The `customer-service` implementation is the following:

```java
@SpringBootApplication
public class CustomerService {

    public static void main(String[] args) {
        SpringApplication.run(CustomerService.class, args);
    }

    @Configuration
    static class CustomerConfig {

        @Bean
        @LoadBalanced
        public RestTemplate restTemplate() {
            return new RestTemplate();
        }

    }

    @RestController
    @Slf4j
    static class CustomerController {

        private static final String TEMPLATE = UriComponentsBuilder.fromUriString("//order-service/orders")
                .queryParam("customerId", "{customerId}").build().toUriString();

        private final RestTemplate restTemplate;
        private final CustomerRepository customerRepository;

        public CustomerController(RestTemplate restTemplate, CustomerRepository customerRepository) {
            this.restTemplate = restTemplate;
            this.customerRepository = customerRepository;
        }

        @GetMapping("/customers/{id}")
        public ResponseEntity<Customer> getCustomer(@PathVariable Long id) {
            log.info("getCustomer with id {}", id);
            Customer customer = customerRepository.getCustomer(id);
            if (customer == null) {
                return new ResponseEntity<>(HttpStatus.NOT_FOUND);
            }
            Order order = restTemplate.getForObject(TEMPLATE, Order.class, id);
            if (order != null) {
                customer.setOrder(new Order(order.getDetails(), order.getTime()));
            }
            return new ResponseEntity<>(customer, HttpStatus.OK);
        }

    }

    @Data
    @AllArgsConstructor
    static class Customer {

        Long id;
        String name;
        Order order;

    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    static class Order {
        String details;
        LocalDateTime time;
    }
}
```

We don't need anymore the `@EnableDiscoveryClient` annotation, when a `DiscoveryClient` implementation (like [Spring Cloud Netflix Eureka](https://cloud.spring.io/spring-cloud-netflix/) used in this post) is available on the classpath the Spring Boot application registers itself with the service registry. (see more details in `EurekaClientAutoConfiguration`) 

By default the Spring Boot application auto-registers itself. There is a handy `/service-registry` actuator endpoint in order to query and register/un-register with the service registry

In our example if we want to put the `order-service` out of service we could use:

```bash
echo '{"status":"OUT_OF_SERVICE"}' | http post :8081/actuator/service-registry
```

After this the `customer-service` will be able to call the `order-service` for about 30 seconds, since this is the default for `Eureka Client`s to look up the registry information to locate their services and make remote calls.  

The `RestTemplate` is no longer created through auto-configuration, in the `customer-service` we need to create one. We use the `@LoadBalanced` annotation to make sure it is load balanced across `order-service` instances. 

### Using Feign

Another approach for invoking downstream services is to use [Feign](https://github.com/OpenFeign/feign) which gives us a type-safe HTTP client. 

```java

@SpringBootApplication
@EnableFeignClients
public class CustomerServiceFeign {

    public static void main(String[] args) {
        SpringApplication.run(CustomerServiceFeign.class, args);
    }

    @RestController
    @Slf4j
    static class CustomerController {

        private final OrderServiceClient orderServiceClient;
        private final CustomerRepository customerRepository;

        public CustomerController(OrderServiceClient orderServiceClient, CustomerRepository customerRepository) {
            this.orderServiceClient = orderServiceClient;
            this.customerRepository = customerRepository;
        }

        @GetMapping("/customers/{id}")
        public ResponseEntity<Customer> getCustomer(@PathVariable Long id) {
            log.info("getCustomer with id {}", id);
            Customer customer = customerRepository.getCustomer(id);
            if (customer == null) {
                return new ResponseEntity<>(HttpStatus.NOT_FOUND);
            }
            Order order = orderServiceClient.getOrder(id);
            if (order != null) {
                customer.setOrder(order.getDetails());
            }
            return new ResponseEntity<>(customer, HttpStatus.OK);
        }

    }

    @FeignClient("order-service")
    interface OrderServiceClient {

        @GetMapping("/orders")
        Order getOrder(@RequestParam("customerId") Long id);
    }
    
}
```

Using the `@FeignClient` annotation on an interface the actual implementation is provisioned at runtime. Under the hood a [Ribbon](https://github.com/Netflix/ribbon) load balancer is created.   

### Spring Boot Admin

[Spring Boot Admin](https://github.com/codecentric/spring-boot-admin) is a great tool to manage your Spring Boot microservices. Unfortunately the Spring Boot / Spring Cloud CLI does not have support for it yet, but we can easily setup a Spring Boot Admin Server.
After creating a Spring Boot application we need to include the following dependencies:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
    <dependency>
       <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

and

```java
@SpringBootApplication
@EnableAdminServer
public class SpringBootAdmin {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootAdmin.class, args);
    }

    @Configuration
    static class WebSecurityConfig extends WebSecurityConfigurerAdapter {

        private final String adminContextPath;

        public WebSecurityConfig(AdminServerProperties adminServerProperties) {
            this.adminContextPath = adminServerProperties.getContextPath();
        }

        @Override
        protected void configure(HttpSecurity http) throws Exception {
            SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
            successHandler.setDefaultTargetUrl(adminContextPath + "/");
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
}
```

<p><img src="/images/2018-08-10/spring-boot-admin.png" alt="Spring Boot Admin" /></p>

### Service Registry on Pivotal Cloud Foundry (PCF)

So far so good. Let's setup this little example on a [Pivotal Web Services](https://run.pivotal.io/) (an instance of PCF hosted by Pivotal) where we will use the Service Registry service (`trial` plan) from [Spring Cloud Services](https://docs.pivotal.io/spring-cloud-services)

First we need to install the [CloudFoundry CLI](https://github.com/cloudfoundry/cli) and create a [Pivotal Web Services](https://run.pivotal.io/) account.

We create a service instance named `service-registry` with the following command

```bash
cf cs p-service-registry trial service-registry
```

Then we need to include the following dependencies into the `Eureka Client` applications (`customer-service`, `customer-service-feign`, `order-service`, `spring-boot-admin`)
  
```xml
<dependency>
    <groupId>io.pivotal.spring.cloud</groupId>
    <artifactId>spring-cloud-services-starter-service-registry</artifactId>
</dependency>
```

It is handy to create `manifest.yml` file for all services. For `customer-service` is 

```yaml
applications:
- name: customer-service
  memory: 756M
  instances: 1
  path: target/customer-service.jar
  buildpack: java_buildpack
  services:
  - service-registry
```

and after building the application with `mvn clean package` we can deploy with 

```bash
cf push --random-route
```

After deploying all the services we should see something like this:

```bash
cf apps

name                     requested state   instances   memory   disk   urls
customer-service         started           1/1         756M     1G     customer-service-shy-mandrill.cfapps.io
customer-service-feign   started           1/1         756M     1G     customer-service-feign-cheerful-hartebeest.cfapps.io
order-service            started           1/1         756M     1G     order-service-appreciative-koala.cfapps.io
spring-boot-admin        started           1/1         756M     1G     spring-boot-admin-anxious-leopard.cfapps.io
```

and we can test the service:

```bash
http customer-service-shy-mandrill.cfapps.io/customers/1
```

```json
{
    "id": 1,
    "name": "Paul Molive",
    "order": {
        "details": "Grilled Chicken Sandwich",
        "time": "2018-08-10T07:32:38.463"
    }
}
```

and the Spring Boot Admin console available at:

<p><img src="/images/2018-08-10/spring-boot-admin2.png" alt="Spring Boot Admin" /></p>

Good practice to set passwords in environment variables. In the Spring Boot Admin we used the following:

```yaml
spring:
  security:
    user:
      name: admin
      password: ${ADMIN_PASSWORD:admin}
``` 

and we set the `ADMIN_PASSWORD` for the `spring-boot-admin` application in the `development` space in Pivotal Cloud Foundry like

```bash
cf set-env spring-boot-admin admin s3cr3t
```

When the `customer-service` was bound the to `service-registry` service instance the connection details were set in the `VCAP_SERVICES` environment variable.

```bash
cf env customer-service
```

```json
{
 "VCAP_SERVICES": {
  "p-service-registry": [
   {
    "binding_name": null,
    "credentials": {
     "access_token_uri": "https://p-spring-cloud-services.uaa.run.pivotal.io/oauth/token",
     "client_id": "p-service-registry-f31ae316-8f1c-4a2d-84ab-02062a0c5aae",
     "client_secret": "ygjAdaV6Gnff",
     "uri": "https://eureka-aa041440-7a75-45b8-bbba-435c79e4ff66.cfapps.io"
    },
    "instance_name": "service-registry",
    "label": "p-service-registry",
    "name": "service-registry",
    "plan": "trial",
    "provider": null,
    "syslog_drain_url": null,
    "tags": [
     "eureka",
     "discovery",
     "registry",
     "spring-cloud"
    ],
    "volume_mounts": []
   }
  ]
 }
}
```

Since we did not configure the the `eureka.client.service-url.defaultZone` how does it get populated from the `VCAP_SERVICES`? The magic is included in the `EurekaServiceConnector` which provides the `eureka.client.*` properties from the `EurekaServiceInfo` which encapsulates the information how to access the `service-registry` service.

All the code examples are on my [github](https://github.com/altfatterz/service-registry)  



