---
layout: post
title: Dead letter queue configuration with RabbitMQ
tags: [RabbitMQ, springboot]
---

In this post we play with dead letter queue configuration with RabbitMQ. A message from a queue can be 'dead-lettered' when one the following things occur:

  * the message is rejected and requeuing is set to false
  * TTL for the message expires
  * the queue length limit is exceeded

For working example
Let's choose the first case with a working example. In a producer app we are going to use `PaymentOrders` as messages which are going to be processed by a consumer.
The `PaymentOrder` message will be rejected when there are insufficient funds on the payer's account.

## The producer

The producer is a Spring Boot application which uses the [Spring AMQP](https://projects.spring.io/spring-amqp/) project to send `PaymentOrder` messages to RabbitMQ.

### The producer's API

The first part of the producer's API is to define the names of the exchange, routing key, incoming and dead letter queue names.

```java
public class Constants {
    public static final String EXCHANGE_NAME = "payment-orders.exchange";
    public static final String ROUTING_KEY_NAME = "payment-orders";
    public static final String INCOMING_QUEUE_NAME = "payment-orders.incoming.queue";
    public static final String DEAD_LETTER_QUEUE_NAME = "payment-orders.dead-letter.queue";
}
```

The second part is to define the message format. We are using JSON in this example. The following json format shows an how we model the `PaymentOrder` which we send to RabbitMQ.

```json
{
  "from":"SA54 22PS JCLV 7LWT 7LHY EBLO",
  "to":"IT23 K545 5414 339G WLPI 2YF6 VBP",
  "amount":54.75
}
```

Note that it is good practice not to use custom serialization format like Java serialization of the payload since that means you need have a java based consumer on the other side.
Good practice is to format the payload in JSON. Every platform and/or language can parse JSON.

### The producer configuration

We need to configure the AMQP infrastructure by declaring the exchange, queue, binding. The dead letter queue configuration is encapsulated in the incoming queue declaration.

There is a concept of `dead letter exchange` (DLX) which is a normal exchange can be any type of `direct`, `topic` or `fanout` exchange. When failure occurs during processing a message fetched from queue, RabbitMQ checks if there is a dead letter exchange configured for that queue.
If there is one configured via `x-dead-letter-exchange` argument then it routes the failed messages to it with the original routing key. This routing key can be overridden via the `x-dead-letter-routing-key` argument.

In this example we are using the `default exchange` (no-name) as the `dead letter exchange` and using the dead letter queue name as the new routing key. There is no need to define any binding for the dead letter queue since every queue is bound to the default exchange by default.

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
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(jackson2JsonMessageConverter());
        return rabbitTemplate;
    }

    @Bean
    public Jackson2JsonMessageConverter jackson2JsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

}
```

We need to redeclare the `RabbitTemplate` since we want to set a JSON message converter.

### The producer logic

In this simple example we just generate random payment orders every 1 second which we send to RabbitMQ for further processing.

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

<p><img src="/images/rabbitmq-dead-letter-queue.png" alt="RabbitMQ dead letter queue" /></p>

## The consumer

### The consumer API

```java
public class Constants {
    public static final String DEAD_LETTER_QUEUE_NAME = "payment-orders.dead-letter.queue";
    public static final String INCOMING_QUEUE_NAME = "payment-orders.incoming.queue";
}

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

The consumer only cares about the queue from where the messages are fetched. The incoming queue must exist otherwise the application will not start. Note that the dead letter queue does not have to exist in order for the consumer to start, but it should exist by the time messages need to be dead-lettered. If it is missing then, the messages will be silently dropped.

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
    public RabbitTemplate rabbitTemplate(ConnectionFactory connectionFactory) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(jackson2JsonMessageConverter());
        return rabbitTemplate;
    }

    @Bean
    public Jackson2JsonMessageConverter jackson2JsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

}
```

### The consumer logic

Whenever a message is available on the incoming queue our `process` method will be invoked with the deserialized `PaymentOrder` instance. Here we simulate the message rejection by throwing an `InsufficientFundsException` randomly.

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

The following shows an example `PaymentOrder` message from the dead letter queue:

<p><img src="/images/rabbitmq-dead-letter-queue-message.png" alt="RabbitMQ dead letter queue message" /></p>


