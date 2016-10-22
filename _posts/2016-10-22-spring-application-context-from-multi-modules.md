---
layout: post
title: Spring application context created from multiple maven modules
tags: [spring]
---

A giant Spring application context created from multiple maven modules can have some challenges. Recently I encountered one at one of my clients. I thought I will describe it with a simple example which you can find on my [github](https://github.com/altfatterz/spring-application-context-from-modules) account.

Let's say you have a maven module which uses Spring XML configuration to define bean definitions, let's call it `legacy-module` from now on. It defines a `GreetingService` with an implementation `DutchGreetingService`. It also defines a `GreetingServiceClient` bean which is using the `GreetingService`
   
```
<bean id="greetingService" class="com.example.service.DutchGreetingService" />
<bean id="greetingServiceClient" class="com.example.service.GreetingServiceClient">
    <constructor-arg ref="greetingService" />
</bean>
```   

Now, this `legacy-module` module is used in another maven module (call it `client-with-java-config` from now on) and provides a different implementation of `GreetingService`. The `client-with-java-config` is a Spring Boot application and as such it is using Spring Java configuration to define the `GreetingService` implementation.  

```
@SpringBootApplication
@ImportResource(value = "classpath:/META-INF/spring/module-context.xml")
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

With the help of `@ImportResource` annotation the bean definitions from the `legacy-module` are imported and together with the `@SpringBootAppliction`'s component scanning feature a combined application context is created.
The only problem with this is that the `HungarianGreetingService` from the `client-with-java-config` is not used, instead it is overridden by the `GreetingService` defined in the `legacy-module`. 

In the logs you can see the following:

```
INFO o.s.b.f.s.DefaultListableBeanFactory : Overriding bean definition for bean 'greetingService' with a different definition: replacing [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=clientWithJavaConfigApp; factoryMethodName=greetingService; initMethodName=null; destroyMethodName=(inferred); defined in com.example.app.ClientWithJavaConfigApp] with [Generic bean: class [com.example.service.DutchGreetingService]; scope=; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in class path resource [META-INF/spring/module-context.xml]]
``` 
 
In order to make it work I had to switch to Spring XML configuration in the client module. (`client-with-xml-config`) and creating the application context explicitly:
 
```
ApplicationContext applicationContext = new ClassPathXmlApplicationContext(
    "classpath:/META-INF/spring/module-context.xml",
    "classpath:/client-context.xml"
);
```
Note that the order of the resources passed as constructor arguments to `ClassPathXmlApplicationContext` matter. Having the `client-context.xml` second with its own `greetingService` defined in it, makes sure that this will be used.
In the logs you will see:

```
INFO o.s.b.f.s.DefaultListableBeanFactory - Overriding bean definition for bean 'greetingService' with a different definition: replacing [Generic bean: class [com.example.service.DutchGreetingService]; scope=; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in class path resource [META-INF/spring/module-context.xml]] with [Generic bean: class [com.example.app.HungarianGreetingService]; scope=; abstract=false; lazyInit=false; autowireMode=0; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=null; factoryMethodName=null; initMethodName=null; destroyMethodName=null; defined in class path resource [client-context.xml]]
```
