---
layout: post
title: AsyncRabbitTemplate
tags: [RabbitMQ, amqp, springboot]
---

Spring AMQP in version 1.6 introduced the `AsyncRabbitTemplate` which allows the caller of send and receive operations not to block. Instead the caller gets a handle to the progress computation in the form of a `ListenableFuture` or via `ListenableFutureCallback`

`ListenableFuture` is a concurrent utility in Spring core and was by inspired by Guava. Basically it is a simple `Future` but it allows you to register callbacks (success or failure callbacks) to be executed once the computation is complete. If the computation has completed then by adding the callbacks they are executed immediately. Although the `ListenableFuture` allows the caller to handle the result later but later by invoking the `get()` method it becomes synchronous by blocking the caller.

```java
public interface ListenableFuture<T> extends Future<T> {

	void addCallback(ListenableFutureCallback<? super T> callback);

	void addCallback(SuccessCallback<? super T> successCallback, FailureCallback failureCallback);

}
```

With the help of `ListenableFutureCallback` you can register a callback which allows the caller to handle the result asynchronously.

```java
public interface ListenableFutureCallback<T> extends SuccessCallback<T>, FailureCallback {}
```

When you pass a message to one of the `send and receive` operations `AsyncRabbitTemplate` calculates a correlation id and together with a `RabbitFuture` instance adds an entry into its internally managed pending futures, returning the reference to the created `RabbitFuture`.
`RabbitFuture` beside being a `ListenableFuture` makes sure that if timeout occurs (it defaults to 30 seconds) then removes itself from the `AsyncRabbitTemplate`s internal registry and completes the future with `AmqpReplyTimeoutException`.








The `AsyncRabbitTemplate` keeps all pending `ListenableFuture`s in a concurrent map using the `correlation id` of the message.


Internally `AsyncRabbitTemplate` uses a `RabbitTemplate` and `SimpleMessageListenerContainer` which can be provided or constructed internally.

When using Spring Boot an instance of `RabbitTemplate` is already auto-configured (see `RabbitAutoConfiguration`), together with a `ConnectionFactory`. So you only need to provide a `SimpleMessageListenerContainer` in order to create an instance of `AsyncRabbitTemplate`.

```java
@Bean
AsyncRabbitTemplate template() {
    SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
    container.setQueueNames(REPLY_QUEUE_NAME);
    return new AsyncRabbitTemplate(rabbitTemplate, container);
}
```

In the above example the first queue the container is configured to listen will be used as the replay queue.




## Publisher confirms and returns




The Spring Amqp project switched to Spring 5 and Java 8 as a foundation in the master branch working towards the 2.0 release.




### Conclusion


