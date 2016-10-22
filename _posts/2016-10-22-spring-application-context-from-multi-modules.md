---
layout: post
title: Spring application context from multi modules
tags: [spring]
---

A giant Spring application context composed created from multiple maven modules can have some challenges. Recently I encountered one at one of my clients, and thought would describe it.

Let's say you have a maven module which uses Spring XML configuration to define bean definitions, lets call it `legacy-module` from now on. It defines a `GreetingService` with an implementation `DutchGreetingService`. It also defines a bean (`GreetingServiceClient`) who is using this `GreetingService`
   
```
<bean id="greetingService" class="com.example.service.DutchGreetingService" />
<bean id="greetingServiceClient" class="com.example.service.GreetingServiceClient">
    <constructor-arg ref="greetingService" />
</bean>
```   

Now, another maven module is depending on this `legacy-module` and provides a different implementation of `GreetingService`. This module is using Spring Boot application, and defines the `GreetingService` with Spring Java configuration.  

```
@SpringBootApplication
@ImportResource(value = "classpath*:/META-INF/spring/module-context.xml")
public class ClientWithJavaConfigApp implements CommandLineRunner {

    @Autowired
    private GreetingServiceClient greetingServiceClient;

    public static void main(String[] args) {
        SpringApplication.run(ClientWithJavaConfigApp.class, args);
    }

    @Override
    public void run(String... args) throws Exception {
        System.out.println(greetingServiceClient.greet("Zoltan"));
    }

    @Bean
    GreetingService greetingService() {
        return new HungarianGreetingService();
    }
}
```

As you can see the `HungarianGreetingService` will be defined using the `greetingService` id from the `@Bean` annotated method name.
The `@SpringBootApplication` is a meta annotation which includes `@ComponentScan` among others. The `@ComponentScan` tells Spring to look for other components in the same package where the bean is defined, allowing it to find the `HungarianGreetingService`.
With the help of `@ImportResource` annotation the bean definitions from the `legacy-module` are imported and together with the `@SpringBootAppliction`'s component scan feature an application context is created.
The only problem with this is that the `GreetingService` from the `legacy-module` is not overridden. 
 
 

```
ApplicationContext applicationContext = new ClassPathXmlApplicationContext(
    "classpath*:/META-INF/spring/module-context.xml",
    "classpath:/client-context.xml"
);

```

