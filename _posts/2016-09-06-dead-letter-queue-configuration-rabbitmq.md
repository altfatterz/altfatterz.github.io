---
layout: post
title: Dead letter queue configuration with RabbitMQ
tags: [RabbitMQ, amqp, springboot]
---

In this blog post I am looking into dead letter queue configuration with RabbitMQ. A message from a queue can be 'dead-lettered' when one the following things occur:

  * the message is rejected and requeuing is set to false
  * TTL for the message expires
  * the queue length limit is exceeded

In order to demonstrate with an example I chose the first case when a message is rejected. The producer is going to to send `PaymentOrders` as messages which are going to be processed by a consumer.
A `PaymentOrder` message will be rejected when there are insufficient funds on the payer's account.

## The producer

The producer is a Spring Boot application which uses the [Spring AMQP](https://projects.spring.io/spring-amqp/) library to send `PaymentOrder` messages to RabbitMQ.

### The producer's API

The first part of the producer's API is to define the name of the exchange, routing key, incoming and dead letter queue.

```java
public class Constants {
    public static final String EXCHANGE_NAME = "payment-orders.exchange";
    public static final String ROUTING_KEY_NAME = "payment-orders";
    public static final String INCOMING_QUEUE_NAME = "payment-orders.incoming.queue";
    public static final String DEAD_LETTER_QUEUE_NAME = "payment-orders.dead-letter.queue";
}
```

The second part is to define the message format. We are using JSON in this example. The following JSON document shows how we model a `PaymentOrder`

```json
{
  "from":"SA54 22PS JCLV 7LWT 7LHY EBLO",
  "to":"IT23 K545 5414 339G WLPI 2YF6 VBP",
  "amount":54.75
}
```

Note that it is good practice not to use custom serialization format like Java serialization of the payload since that means you need to have a java based consumer. Good practice is to format the payload in JSON. Every platform and/or language can parse JSON.

### The producer configuration

We need to configure the AMQP infrastructure. The dead letter queue configuration is encapsulated in the incoming queue declaration.

There is a concept of [dead letter exchange](https://www.rabbitmq.com/dlx.html) (DLX) which is a normal exchange of type `direct`, `topic` or `fanout`. When failure occurs during processing a message fetched from a queue, RabbitMQ checks if there is a dead letter exchange configured for that queue. If there is one configured via `x-dead-letter-exchange` argument then it routes the failed messages to it with the original routing key. This routing key can be overridden via the `x-dead-letter-routing-key` argument.

In this example we are using the `default exchange` (no-name) as the `dead letter exchange` and using the dead letter queue name as the new routing key. This will work since any queue is bound to the default exchange with the binding key equal to the queue name.

```java
@Configuration
public class AmqpConfig {

    @Bean
    DirectExchange exchange() {
        return new DirectExchange(Constants.EXCHANGE_NAME);
    }

    @Bean
    Queue incomingQueue() {
        return QueueBuilder.durable(Constants.INCOMING_QUEUE_NAME)
                .withArgument("x-dead-letter-exchange", "")
                .withArgument("x-dead-letter-routing-key", Constants.DEAD_LETTER_QUEUE_NAME)
                .build();
    }

    @Bean
    Binding binding() {
        return BindingBuilder.bind(incomingQueue()).to(exchange()).with(Constants.ROUTING_KEY_NAME);
    }

    @Bean
    Queue deadLetterQueue() {
        return QueueBuilder.durable(Constants.DEAD_LETTER_QUEUE_NAME).build();
    }

    @Bean
    public Jackson2JsonMessageConverter jackson2JsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

}
```

The builder API for queues and exchanges are pretty convenient and is available starting with the 1.6 version of the Spring AMQP library.

In the RabbitMQ management console the `DLX` and `DLK` labels indicate that the `dead letter exchange` and `dead letter routing key` arguments are set on the incoming queue.

<p><img src="/images/rabbitmq-dead-letter-queue.png" alt="RabbitMQ dead letter queue" /></p>

### The producer logic

The producer generates random `PaymentOrder` messages every 5 second which are sent to RabbitMQ for further processing. Spring’s `AmqpTemplate` is auto-configured and it can be wired into our component. Since the message format is JSON the `Jackson2JsonMessageConverter` is defined which will be associated automatically to the auto-configured `AmqpTemplate`.

```java
@Component
public class Producer {

    private AmqpTemplate amqpTemplate;

    public Producer(AmqpTemplate amqpTemplate) {
        this.amqpTemplate = amqpTemplate;
    }

    @Scheduled(fixedDelay = 1000L)
    public void send() {

        PaymentOrder paymentOrder = new PaymentOrder(
                Iban.random().toFormattedString(),
                Iban.random().toFormattedString(),
                new BigDecimal(1D + new Random().nextDouble() * 100D).setScale(2, BigDecimal.ROUND_FLOOR));

        amqpTemplate.convertAndSend(Constants.EXCHANGE_NAME, Constants.ROUTING_KEY_NAME, paymentOrder);
    }
}

```

## The consumer

For this simple example the consumer is also a Spring Boot application, but in real applications the consumer and producer don't necessary are on the same platform/language.

### The consumer API

The first part of the consumers's API is to specify to which queue it is connected to.

```java
public class Constants {
    public static final String DEAD_LETTER_QUEUE_NAME = "payment-orders.dead-letter.queue";
    public static final String INCOMING_QUEUE_NAME = "payment-orders.incoming.queue";
}
```

The second part is to adapt to the message format which was defined by the producer. Note that in this case both applications are Java based, so I could have created a jar file containing the `PaymentOrder` class file and share it with consumer and producer. However this is bad practice since it introduces tight coupling based on a shared library. Better approach is to use a bit of code duplication (`PaymentOrder` class in this case) and use a more loose coupling approach by agreeing on the message format.

```java
public class PaymentOrder {

    String from;
    String to;
    BigDecimal amount;

    @JsonCreator
    public PaymentOrder(@JsonProperty("from") String from,
                        @JsonProperty("to") String to,
                        @JsonProperty("amount") BigDecimal amount) {
        this.from = from;
        this.to = to;
        this.amount = amount;
    }

    // getters and toString()
}
```

### The consumer configuration

The consumer only cares about the queue from where the messages are fetched. The incoming queue must exist otherwise the consumer will not start. Note that the `dead letter queue` does not have to exist in order for the consumer to start, but it should exist by the time messages need to be 'dead-lettered'. If it is missing then, the messages will be silently dropped.

```java
@Configuration
public class AmqpConfig {

    @Bean
    Queue incomingQueue() {
        return QueueBuilder.durable(Constants.INCOMING_QUEUE_NAME)
                .withArgument("x-dead-letter-exchange", "")
                .withArgument("x-dead-letter-routing-key", Constants.DEAD_LETTER_QUEUE_NAME)
                .build();
    }

    @Bean
    public Jackson2JsonMessageConverter jackson2JsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

}
```

By default the requeuing is enabled. In order to 'dead-letter' messages you need to set the following property to false.

```yml
spring:
  rabbitmq:
    listener:
      default-requeue-rejected: false
```

However if you would like to enable requeuing in some error scenarios is better to leave the requeuing enabled and leverage the `AmqpRejectAndDontRequeueException` which will send the `basic.reject` with requeue=false.

### The consumer logic

Whenever a message is available on the incoming queue the `process` method will be invoked with the deserialized `PaymentOrder` instance. Here we simulate the message rejection by throwing an `InsufficientFundsException` which extends the `AmqpRejectAndDontRequeueException` exception.

```java
@Component
public class Consumer {

    @RabbitListener(queues = Constants.INCOMING_QUEUE_NAME)
    public void process(@Payload PaymentOrder paymentOrder) throws InsufficientFundsException {
        if (new Random().nextBoolean()) {
            throw new InsufficientFundsException("insufficient funds on account " + paymentOrder.getFrom());
        }
    }

}
```

The following image shows an example of a `PaymentOrder` message which was rejected and as a result ended up in the `dead letter queue`

<p><img src="/images/rabbitmq-dead-letter-queue-message.png" alt="RabbitMQ dead letter queue message" /></p>

Sometimes it helps to automatically retry a failed operation in case it might succeed on a subsequent attempt. The Spring AMQP library provides support for this with the help of `RetryTemplate` which is part of the [Spring Retry](https://github.com/spring-projects/spring-retry) project (was pulled out from Spring Batch).
Spring Boot makes it super easy to configure the `RetryTemplate` as shown the the following example.

```yml
spring:
  rabbitmq:
    listener:
      retry:
        enabled: true
        initial-interval: 2000
        max-attempts: 2
        multiplier: 1.5
        max-interval: 5000
```

With the above configuration the retry functionality is enabled (disabled by default), there should be maximum 2 attempts to deliver the message, between the first and the second attempt should be 2 seconds, later with a multiplier of 1.5 to the previous retry interval and up to 5 seconds.
Running the consumer you will see in the logs

```
2016-09-07 21:56:53.396  INFO 11995 --- [cTaskExecutor-1] com.example.consumer.Consumer            : Processing at 'Wed Sep 07 21:56:53 CEST 2016' payload 'PaymentOrder{from='RS32 5346 0536 6006 4886 88', to='FI61 8364 3364 9834 16', amount=45.57}'
2016-09-07 21:56:55.399  INFO 11995 --- [cTaskExecutor-1] com.example.consumer.Consumer            : Processing at 'Wed Sep 07 21:56:55 CEST 2016' payload 'PaymentOrder{from='RS32 5346 0536 6006 4886 88', to='FI61 8364 3364 9834 16', amount=45.57}'
2016-09-07 21:56:55.401  WARN 11995 --- [cTaskExecutor-1] o.s.a.r.r.RejectAndDontRequeueRecoverer  : Retries exhausted for message (Body:'{"from":"RS32 5346 0536 6006 4886 88","to":"FI61 8364 3364 9834 16","amount":45.57}' MessageProperties [headers={__TypeId__=com.example.producer.api.PaymentOrder}, timestamp=null, messageId=null, userId=null, receivedUserId=null, appId=null, clusterId=null, type=null, correlationId=null, correlationIdString=null, replyTo=null, contentType=application/json, contentEncoding=UTF-8, contentLength=0, deliveryMode=null, receivedDeliveryMode=PERSISTENT, expiration=null, priority=0, redelivered=false, receivedExchange=payment-orders.exchange, receivedRoutingKey=payment-orders, receivedDelay=null, deliveryTag=31, messageCount=0, consumerTag=amq.ctag-vd18OXS9PSOeJmBQLY4o-w, consumerQueue=payment-orders.incoming.queue])
```

### Conclusion

As you could see the dead letter queue configuration is pretty simple using RabbitMQ. The example used in this blog post is available on [my GitHub account](https://github.com/altfatterz/rabbit-dead-letter-queue).