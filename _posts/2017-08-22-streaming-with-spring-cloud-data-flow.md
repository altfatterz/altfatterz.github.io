---
layout: post
title: Streaming with Spring Cloud Data Flow
tags: [springcloud]
---

In this blog post I wanted to create the "Hello World" streaming example using Spring Cloud Data Flow which helps to create a microservices architecture for ingesting, transforming and storing data.   
Spring Cloud Data Flow originates from Spring XD which was standalone project to build distributed data pipelines for real-time and batch processing.

In contrast to Spring XD, in Spring Cloud Data Flow the modules are autonomous deployable Spring Boot apps, which can be run individually with java -jar. 
These Spring Boot apps can use Spring Cloud Stream to quickly build a message-driven microservice, where the messaging middleware (like Apache Kafka, RabbitMQ, etc) is abstracted away.
In the Spring Cloud Stream terminology streams are made up `sources`, `sinks` and optionally `processors`.

Below you can see a simple `source` application which with the help of `InputChannelAdapter` annotation sends a message to the output channel of the `source` using an auto-configured poller which can be further customized using the `spring.integration.poller` property.
The `EnableBinding` annotation triggers the Spring Cloud Stream infrastructure.

```java
@EnableBinding(Source.class)
@SpringBootApplication
public class SourceApp {

    public static void main(String[] args) {
        SpringApplication.run(SourceApp.class, args);
    }

    @InboundChannelAdapter(value = Source.OUTPUT)
    public String source() {
        return new Date().toString();
    }
}
```


```java
@EnableBinding(Sink.class)
@SpringBootApplication
@Slf4j
public class SinkApp {

    public static void main(String[] args) {
        SpringApplication.run(SinkApp.class, args);
    }

    @ServiceActivator(inputChannel = Sink.INPUT)
    public void logger(String payload) {
        log.info("received {}", payload);
    }
}
```

```java
@SpringBootApplication
@EnableBinding(Processor.class)
@Slf4j
public class ProcessorApp {

    public static void main(String[] args) {
        SpringApplication.run(ProcessorApp.class, args);
    }

    @ServiceActivator(inputChannel = Processor.INPUT, outputChannel = Processor.OUTPUT)
    public String transform(String payload) {
        log.info("Processor received {}", payload);
        return payload + " processed";
    }
}
```

#### Register the applications

```bash
dataflow:>app register --name source-app --type source --uri maven://com.example:source-app:jar:0.0.1-SNAPSHOT
Successfully registered application 'source:source-app'
dataflow:>app register --name processor-app --type processor --uri maven://com.example:processor-app:jar:0.0.1-SNAPSHOT
Successfully registered application 'processor:processor-app'
dataflow:>app register --name sink-app --type sink --uri maven://com.example:sink-app:jar:0.0.1-SNAPSHOT
Successfully registered application 'sink:sink-app'
dataflow:>app list
╔══════════╤═════════════╤════════╤════╗
║  source  │  processor  │  sink  │task║
╠══════════╪═════════════╪════════╪════╣
║source-app│processor-app│sink-app│    ║
╚══════════╧═════════════╧════════╧════╝
```

<p><img src="/images/2017-08-22/scdf-apps.png" alt="Spring Cloud Data Flow - Apps" /></p>

#### Create a stream
```bash
dataflow:>stream create --name myStream --definition 'source-app | processor-app | sink-app'
Created new stream 'myStream'
dataflow:>stream list
╔═══════════╤═════════════════════════════════════╤══════════════════════════════════════════════════════════════════════╗
║Stream Name│          Stream Definition          │                                Status                                ║
╠═══════════╪═════════════════════════════════════╪══════════════════════════════════════════════════════════════════════╣
║myStream   │source-app | processor-app | sink-app│The app or group is known to the system, but is not currently deployed║
╚═══════════╧═════════════════════════════════════╧══════════════════════════════════════════════════════════════════════╝
```

<p><img src="/images/2017-08-22/scdf-streams.png" alt="Spring Cloud Data Flow - Streams" /></p>

#### Deploy the stream
```bash
dataflow:>stream deploy --name myStream
Deployment request has been sent for stream 'myStream'
dataflow:>stream list
╔═══════════╤═════════════════════════════════════╤════════════════════════════════════════╗
║Stream Name│          Stream Definition          │                 Status                 ║
╠═══════════╪═════════════════════════════════════╪════════════════════════════════════════╣
║myStream   │source-app | processor-app | sink-app│All apps have been successfully deployed║
╚═══════════╧═════════════════════════════════════╧════════════════════════════════════════╝
```

<p><img src="/images/2017-08-22/scdf-runtime.png" alt="Spring Cloud Data Flow - Runtime" /></p>

```bash
brew install rabbitmq
```

```bash
brew services start rabbitmq
```

```
maven://<groupId>:<artifactId>[:<extension>[:<classifier>]]:<version>
```

Register multiple apps at one time, you can store them in a properties file with <type>.<name>
```bash
stream.source-app=
stream.processor-app=
stream.sink-app=
```

and then you can import them via:

```
dataflow:>app import --uri file:///tmp/my-stream-apps.properties
```
Look into: https://github.com/spring-cloud/spring-cloud-dataflow/blob/master/spring-cloud-dataflow-docs/src/main/asciidoc/streams.adoc