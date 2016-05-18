---
layout: post
title: Application events with Spring
tags: [springframework]
---

In this post I am looking into the awesome application event support provided by Spring Framework,
which starting from version 4.2 got even better by introducing an annotation model to consume events, the possibility to publish any object as event not forcing to extend from `ApplicationEvent`.

Application events are less used in real applications, but internally in Spring framework (`ContextRefreshedEvent`, `RequestHandledEvent`, etc...), but also in Spring Boot (`ApplicationStartedEvent`, `ApplicationEnvironmentPreparedEvent`, etc ...) are used intensively.

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
    public TodoCreatedEventProducer(ApplicationEventPublisher publisher) { ... }

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

    private T source;

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

Note that this change will be global to the `ApplicationContext` meaning that all methods annotated with `@EventListener` will be executed asynchronously. If we like to have some events delivered synchronously others asynchronously within the same `ApplicationContext` check out [this](https://www.keyup.eu/en/blog/101-synchronous-and-asynchronous-spring-events-in-one-application) blog post which details how it can be done with a custom annotation and a custom `ApplicationEventMulticaster` wrapping a synchronous and asynchronous `ApplicationEventMulticaster` instance.

> A much easier way to handle some events asynchronously is to use the `@Async` annotation.

```java
@Async
@EventListener
void handleAsync(MedicalRecordUpdatedEvent event) {
    // MedicalRecordUpdatedEvent is processed in a separate thread.
}
```

### Filtering

It is possible to filter the events in the listener via the `condition` attribute. The following example shows that the event listener is called only if the bid is higher or equal than 100.

```java
@EventListener(condition = "#bidCreatedEvent.amount >= 100")
public void handleHighBids(BidCreatedEvent bidCreatedEvent) {
    ...
}
```

Note that starting from Spring 4.3.0.RC1 we are able to specify the condition to refer to beans (e.g. @beanName.method()).

### Transaction bound events

With synchronous event handling the listener can be bound to a phase of the transaction in which the publisher is running. The following example shows that the listener should only handle the `TaskScheduledEvent` once the transaction in which it was published committed successfully.

```java
@TransactionalEventListener
public void handleAfterCommit(TaskScheduledEvent event)
    ...
}
```

If no transaction is running, the listener is not invoked at all, but we can override this with `fallbackExecution` attribute setting it to true. With the `phase` attribute we can bound to other phases of a transaction (`BEFORE_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION`, `AFTER_COMMIT` (default)). In the example below the listener method will be executed if the transaction in which the publisher is running was rolled back.

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
public void handleAfterRollback(TaskScheduledEvent event) {
    ...
}
```

### Conclusion

As we have seen the support for application events in Spring is pretty comprehensive. It can be very helpful in event driven business applications. If you would like to try out these features have a look at this [https://github.com/altfatterz/application-events-with-spring](https://github.com/altfatterz/application-events-with-spring) repository.


