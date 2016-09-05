---
layout: post
title: Dead letter queue configuration with RabbitMQ
tags: [RabbitMQ, springboot]
---

In this post I am looking into dead letter queue configuration with RabbitMQ. A message from a queue can be 'dead-lettered' when one the following things occur:

    * the message is rejected and requeuing is set to false
    * TTL for the message expires
    * the queue length limit is exceeded

Let's choose the first case with a working example. In a producer app we are going to use `PaymentOrders` as messages which are going to be processed by a consumer.
The `PaymentOrder` message will be rejected when there are insufficient funds on the payer's account.

## The producer

The producer is a Spring Boot application which uses the [Spring AMQP](https://projects.spring.io/spring-amqp/) project to send `PaymentOrder` messages to RabbitMQ.

### The producer API

The first part of the producer's API is to define the exchange name where the messages are going to be sent.

```java
public class Constants {
    public static final String EXCHANGE_NAME = "payment-orders.exchange";
    public static final String INCOMING_QUEUE_NAME = "payment-orders.incoming.queue";
    public static final String DEAD_LETTER_QUEUE_NAME = "payment-orders.dead-letter.queue";
}
```

The second part is to define the message format. We are using JSON in the example. The following example shows an example `PaymentOrder` sent to RabbitMQ.

```json
{
  "from":"SA54 22PS JCLV 7LWT 7LHY EBLO",
  "to":"IT23 K545 5414 339G WLPI 2YF6 VBP",
  "amount":54.75
}
```

### The producer configuration

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

### The producer logic

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

### The consumer logic


```java
@Component
public class Consumer {

    private static final Logger logger = LoggerFactory.getLogger(Consumer.class);

    @RabbitListener(queues = Constants.INCOMING_QUEUE_NAME)
    public void process(@Payload PaymentOrder paymentOrder) throws InsufficientFundsException {
        logger.info("Processing payload \'{}\'", paymentOrder);

        if (new Random().nextBoolean()) {
            throw new InsufficientFundsException("insufficient funds on account " + paymentOrder.getFrom());
        }
    }
}
```

<p><img src="/images/rabbitmq-dead-letter-queue-message.png" alt="RabbitMQ dead letter queue message" /></p>

<p><img src="/images/rabbitmq-dead-letter-queue.png" alt="RabbitMQ dead letter queue" /></p>
