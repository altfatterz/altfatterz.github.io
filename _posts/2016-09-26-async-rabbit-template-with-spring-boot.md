---
layout: post
title: AsyncRabbitTemplate with Spring Boot
tags: [RabbitMQ, amqp, springboot]
---

Spring AMQP in version 1.6 introduced the `AsyncRabbitTemplate` which allows the caller of the send and receive operations (sendAndReceive, convertSendAndReceive) not to block. The caller instead gets a handle to the computation in progress in the form of a `ListenableFuture` or can handle the result in a callback via `ListenableFutureCallback`

`ListenableFuture` is a concurrent utility in Spring core and was by inspired by Guava. Basically it is a simple `Future` but it allows you to register callbacks (success or failure callbacks) to be executed once the computation is complete. If the computation has completed then by adding the callbacks they are executed immediately. Although the `ListenableFuture` allows the caller to handle the result later but by invoking the `get()` method it becomes synchronous by blocking the caller.

With the help of `ListenableFutureCallback` you can register a callback which allows the caller to handle the result asynchronously.

```java
public interface ListenableFuture<T> extends Future<T> {

	void addCallback(ListenableFutureCallback<? super T> callback);

	void addCallback(SuccessCallback<? super T> successCallback, FailureCallback failureCallback);

}
```


```java
public interface ListenableFutureCallback<T> extends SuccessCallback<T>, FailureCallback {}
```

When you pass a payload to one of the send and receive operations `AsyncRabbitTemplate` calculates a correlation id and together with a `RabbitFuture` instance forms an entry which is put into its internally managed pending futures `ConcurrentMap`, returning the reference to the created `RabbitFuture`.

`RabbitFuture` is the base class of `ListenableFuture` in AsyncRabbitTemplate. It makes sure if the response is not received within the configured receive timeout (it defaults to 30 seconds) it cancels the computation, removes itself from the `AsyncRabbitTemplate`s internal `ConcurrentMap` and completes the future with `AmqpReplyTimeoutException`.

`AsyncRabbitTemplate` leverages a `RabbitTemplate` to delegate the send and receive operations and a `SimpleMessageListenerContainer` to fetch the responses from the reply queue. Both can be provided externally or constructed internally.

When using Spring Boot an instance of `RabbitTemplate` is already auto-configured (see `RabbitAutoConfiguration`), together with a `ConnectionFactory`. So you only need to provide a `SimpleMessageListenerContainer` in order to create an instance of `AsyncRabbitTemplate`.

```java
@Bean
AsyncRabbitTemplate template() {
    SimpleMessageListenerContainer container = new SimpleMessageListenerContainer(connectionFactory);
    container.setQueueNames(REPLY_QUEUE_NAME);
    return new AsyncRabbitTemplate(rabbitTemplate, container);
}
```

In the above example the first queue the container is configured to listen will be used as the reply queue.

If you fancy to try out these things look into this [https://github.com/altfatterz/async-rabbit-template](https://github.com/altfatterz/async-rabbit-template) repository.

