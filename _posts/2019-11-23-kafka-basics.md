---
layout: post
title: Kafka Basics
tags: [kafka]
---

In this post we are looking into Apache Kafka. 

#### Install Kafka

There are many different ways to install Kafka. We can get the binary from [https://kafka.apache.org/downloads](https://kafka.apache.org/downloads) 
or we can use Docker. The fastest way on Mac is installing with Homebrew. 

```bash
$ brew install kafka
```

The above will also install Zookeeper.

Homebrew will install the configuration files into `/usr/local/etc/kafka/` directory. I prefer to create my own copy of these files.
They are included into the [https://github.com/altfatterz/learning-kafka](https://github.com/altfatterz/learning-kafka) 

```bash
$ git clone https://github.com/altfatterz/learning-kafka
```   

After cloning the repository we need to change the followings:
 
* the `log.dirs` property in the `config/server.properties` to the `<learning-kafka-path>/data/kafka` 
* the `dataDir` property in the `config/zookeeper.properties` to the value `<learning-kafka-path>/data/zookeeper`

#### Start Zookeeper and Kafka

Now we can start Zookeeper and Kafka in separate terminals

```bash
$ zookeeper-server-start config/zookeeper.properties
$ kafka-server-start config/server.properties
```

In the logs we should see

```bash
[2019-12-23 16:09:43,145] INFO [KafkaServer id=0] started (kafka.server.KafkaServer)
```

#### Kafka Topics

We create a `demo-topic` with 3 partitions and 1 replication factor.

```bash
$ kafka-topics --bootstrap-server localhost:9092 --topic demo-topic --create --partitions 3 --replication-factor 1
```

We can verify the result using:

```bash
$ kafka-topics --bootstrap-server localhost:9092 --topic demo-topic --describe

Topic:demo-topic	PartitionCount:3	ReplicationFactor:1	Configs:segment.bytes=1073741824
	Topic: demo-topic	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	Topic: demo-topic	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	Topic: demo-topic	Partition: 2	Leader: 0	Replicas: 0	Isr: 0
```

To delete a topic we can use:

```bash
$ kafka-topics --bootstrap-server localhost:9092 --topic demo-topic --delete
```

And to see all the topics we use:

```bash
$ kafka-topics --bootstrap-server localhost:9092 --topic demo-topic --list
```

Note that the previously used `--zookeeper` option is deprecated now, instead we can use the `--bootstrap-server` option.
  
#### Kafka Console Consumer

To start a simple consumer we can use the `kafka-console-consumer` command

```bash
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic demo-topic
```

It does not print anything yet since there are no messages in the topic.

#### Kafka Console Producer

In another terminal we produce some messages with the `kafka-console-producer` command

```bash
$ kafka-console-producer --broker-list 127.0.0.1:9092 --topic demo-topic
>first message
>second message
>third message
>^C
```

In the terminal where the `kafka-console-consumer` is started we should now see the messages.

#### Kafka Consumer Groups

By default if we don't specify a `group` for the consumer a `consumer group` will be generated with a single member   

```bash
$ kafka-consumer-groups --bootstrap-server localhost:9092 --list
console-consumer-75696
```

We can specify a group when defining the consumer. 

```bash
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic demo-topic --group demo-app
```

Let's start another `kafka-console-consumer` with the above command in another terminal with the same `demo-app` group.  

We can monitor the `current offset` and `lag` of the consumers connected to the partitions.

```bash
$ kafka-consumer-groups --bootstrap-server localhost:9092 --group demo-app --describe

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
demo-app        demo-topic      0          1               1               0               consumer-1-7e3aa047-c981-4be1-bfac-c0e90305de12 /192.168.1.6    consumer-1
demo-app        demo-topic      1          2               2               0               consumer-1-7e3aa047-c981-4be1-bfac-c0e90305de12 /192.168.1.6    consumer-1
demo-app        demo-topic      2          2               2               0               consumer-1-a1cf8bb1-5aef-4e09-8d88-d3bdc2ff6f33 /192.168.1.6    consumer-1
```

Here we can see that one consumer is connected to the first two partitions while the other consumer is connected to the third partition.

#### KafkaConsumer

To connect with Java to a topic we need the following dependency:

```xml
<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>${kafka.version}</version>
</dependency>
```

We set the following properties to create a `KafkaConsumer` instance

```bash
Properties properties = new Properties();
properties.setProperty(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
properties.setProperty(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.setProperty(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
properties.setProperty(ConsumerConfig.GROUP_ID_CONFIG, "demo-app");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties);
```

Then we subscribe to the `demo-topic` topic and poll the consumer for a duration and try again.

```java
// subscribe consumer to topic
consumer.subscribe(Arrays.asList("demo-topic"));

// poll for new data
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(200));
    for (ConsumerRecord<String, String> record : records) {
        logger.info("Key: {}, Value:{}, Partition: {}, Offset: {}", record.key(), record.value(), record.partition(), record.offset());
    }
}
```

Next if we produce the `Kafka rocks!` message with the `kafka-console-producer` we can see in the logs of our Java based Kafka consumer:

```bash
12:04:53.221 [main] INFO  c.g.altfatterz.KafkaConsumerDemo - Key: null, Value:Kafka rocks!, Partition: 2, Offset: 4
``` 

#### KafkaProducer

We can also produce messages with Java using the `KafkaProducer` API

```java
// producer properties
Properties properties = new Properties();
properties.setProperty(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
properties.setProperty(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
properties.setProperty(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

// producer
KafkaProducer<String, String> producer = new KafkaProducer<>(properties);
```

```bash
// create a producer record
ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC, "learning kafka");

logger.info("send message asynchronously");
producer.send(record);

logger.info("flushing and closing the producer");
producer.close();
```

#### Conclusion

In this post we looked into 

1. Starting Zookeeper and Kafka
2. Useful CLI commands: `kafka-topics`, `kafka-console-consumer`, `kafka-console-producer`, `kafka-consumer-groups`
3. The Kafka Java API using `KafkaConsumer` and `KafkaProducer`

Source code is available here: [https://github.com/altfatterz/learning-kafka](https://github.com/altfatterz/learning-kafka)