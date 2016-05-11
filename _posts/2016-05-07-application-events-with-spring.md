---
layout: post
title: Application events with Spring
tags: [springframework]
---

In this post I would like to describe the awesome application event support provided by Spring Framework,
which starting from version 4.2 got even better by introducing an annotation model to consume events, the possibility to publish any object as event not forcing to extend from `ApplicationEvent`.

Application events are less used in real applications, but internally in Spring framework is used intensively. (see `ContextRefreshedEvent`, `RequestHandledEvent`, etc...)
Basically the Spring `ApplicationContext` is capable to behave like an event bus which enables simple communication between Spring beans within the same `ApplicationContext`

This is how we could define a consumer of `TodoCreatedEvent`s.

```java
@Component
class TodoCreatedEventListener {

    @EventListener
    void handle(TodoCreatedEvent event) {
        ...
    }
}
```

`@EventListener` is a core annotation, no extra configuration is needed using Java config. Internally the [`EventListenerMethodProcessor`](https://github.com/spring-projects/spring-framework/blob/master/spring-context%2Fsrc%2Fmain%2Fjava%2Forg%2Fspringframework%2Fcontext%2Fevent%2FEventListenerMethodProcessor.java) registers an `ApplicationListener` instance with the event type inferred from the method signature.

The `TodoCreatedEvent` can be an ordinary POJO. Internally it will be wrapped into a `PayloadApplicationEvent`.

```java
class TodoCreatedEvent {

  private String title;

  public TodoCreatedEvent(String title) {
     this.title = title;
  }
```

On the producer side the `ApplicationEventPublisher` was extended to publish any POJO as an event.

```java
@Component
class TodoCreatedEventProducer {

    private final ApplicationEventPublisher publisher;

    @Autowired
    public MyComponent(ApplicationEventPublisher publisher) { ... }

    public void createTodo(Todo todo) {
        publisher.publishEvent(new TodoCreatedEvent(todo.getTitle()));
    }

}
```

### Generic events

It is possible to define the events using generics

```java
@EventListener
public void onBidCeated(EntityCreatedEvent<Bid> event) {
    ...
}
```

When publishing the event in order for this to work we have two options. We could resolve the generic parameter, something like:

```java
class BidCreatedEvent extends EntityCreatedEvent<Bid> { ... }
```

Or we could implement [`ResolvableTypeProvider`](https://github.com/spring-projects/spring-framework/blob/master/spring-core%2Fsrc%2Fmain%2Fjava%2Forg%2Fspringframework%2Fcore%2FResolvableTypeProvider.java) to help Spring to figure out if the event instance matches a generic signature.

```java
class EntityCreatedEvent<T> implements ResolvableTypeProvider {

    private Object source;

    public EntityCreatedEvent(T source) {
        this.source = source;
    }

    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(),
                ResolvableType.forInstance(source));
    }
}
```

### Async events

By default event listeners receive events synchronously, meaning that the publishing thread will block until all listeners have finished processing the event. The advantage of this is that if the publisher is running in a transactional context, the listener will receive the event within the same transactional context.
However if processing events takes long time and scaling is important we can tell Spring to handle events asynchronously. In order to do this we need to redefine the `ApplicationEventMulticaster` bean with id `applicationEventMulticaster` configuring it with an asynchronous `TaskExecutor`.


```java
@Bean
ApplicationEventMulticaster applicationEventMulticaster() {
    SimpleApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster();
    eventMulticaster.setTaskExecutor(new SimpleAsyncTaskExecutor());
    eventMulticaster.setErrorHandler(TaskUtils.LOG_AND_SUPPRESS_ERROR_HANDLER);
    return eventMulticaster;
}
```

Note that this change will be global to the `ApplicationContext` meaning that all methods annotated with `@EventListener` will be executed asynchronously. If you like to have some events delivered synchronously others asynchronously within the same `ApplicationContext` check out [this](https://www.keyup.eu/en/blog/101-synchronous-and-asynchronous-spring-events-in-one-application) blog post which details how it can be done with a custom annotation and a custom `ApplicationEventMulticaster` wrapping a synchronous and asynchronous `ApplicationEventMulticaster` instances.

### Filtering

It is possible to filter the events in the listener via the `condition` attribute

### Transaction bound event

Spring documentation
17.8 Transaction bound event


Reources:

https://www.keyup.eu/en/blog/101-synchronous-and-asynchronous-spring-events-in-one-application
https://dzone.com/articles/eventing-spring-framework
http://howtodoinjava.com/spring/spring-core/how-to-publish-and-listen-application-events-in-spring/