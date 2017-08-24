---
layout: post
title: Streaming with Spring Cloud Data Flow
tags: [springcloud]
---

In this blog post I want to show how to create a "Hello World" streaming example using Spring Cloud Data Flow which simplifies the development and deployment of applications focused on data processing use-cases.
Spring Cloud Data Flow originates from Spring XD which was a standalone project to build distributed data pipelines for real-time and batch processing.

In contrast to Spring XD, in Spring Cloud Data Flow the modules are autonomous deployable Spring Boot apps, which can be run individually with java -jar. 
These Spring Boot apps can use Spring Cloud Stream to quickly build a message-driven microservice, where the messaging middleware (like Apache Kafka, RabbitMQ, etc) is abstracted away.
In `Spring Cloud Stream` terminology streams are made up of `sources`, `sinks` and optionally `processors`.

Below you can see a simple `source` application which with the help of `@InputChannelAdapter` annotation sends a message every second to the output channel of the `source` using an auto-configured poller which can be further customized using the `spring.integration.poller` property.
The `@EnableBinding` annotation triggers the Spring Cloud Stream infrastructure.

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

In the `sink` application with the help `@StreamListener` annotation the application receives messages from the input channel of the `sink`. 

```java
@EnableBinding(Sink.class)
@SpringBootApplication
@Slf4j
public class SinkApp {

    public static void main(String[] args) {
        SpringApplication.run(SinkApp.class, args);
    }

    @StreamListener(Sink.INPUT)
    public void logger(String payload) {
        log.info("received {}", payload);
    }
}

```

And finally there is a simple `processor` application which transforms the received messages. 

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

The parameter specified in the `@EnableBinding` annotation (`Sink`, `Source`, `Processor`) are defined in Spring Cloud Stream. They are for convenience and they use the `@Input` annotation to define an input channel (through which received messages enter the application) and use the `@Output` annotation to define an output channel (through which published messages leave the application). 
The user can define its own interface specified at `@EnableBinding` annotation.

### Running the applications without Spring Cloud Data Flow

In this example I am using RabbitMQ as a messaging middleware. Binding properties needs to be set in order for the applications to know which `exchange` send to or receive from the messages.

```bash
java -jar target/sink-app-0.0.1-SNAPSHOT.jar \ 
    --spring.cloud.stream.bindings.input.destination: demo-processed
java -jar target/processor-app-0.0.1-SNAPSHOT.jar \
    --spring.cloud.stream.bindings.input.destination: demo \
    --spring.cloud.stream.bindings.output.destination: demo-processed
java -jar target/source-app-0.0.1-SNAPSHOT.jar \
    --spring.cloud.stream.bindings.output.destination: demo 
```   

Here it is specified that messages produced by the `source-app` should be sent to the `demo` exchange, from where the `processor-app` should take the messages, processes them and should put them on the `demo-processed` exchange from where finally the `sink-app` should take the messages and print them out. 

After starting first RabbitMQ and then all the three applications you should see the that the `sink-app` receives the processed messages:

```bash
2017-08-24 16:38:44.036  INFO 5850 --- [           main] s.b.c.e.t.TomcatEmbeddedServletContainer : Tomcat started on port(s): 8082 (http)
2017-08-24 16:38:44.038  INFO 5850 --- [           main] com.example.sinkapp.SinkApp              : Started SinkApp in 13.578 seconds (JVM running for 14.81)
2017-08-24 16:39:23.712  INFO 5850 --- [u2IdLMiby6OzA-1] com.example.sinkapp.SinkApp              : received Thu Aug 24 16:39:23 CEST 2017 processed
2017-08-24 16:39:24.581  INFO 5850 --- [u2IdLMiby6OzA-1] com.example.sinkapp.SinkApp              : received Thu Aug 24 16:39:24 CEST 2017 processed
...
```

### Running the applications with Spring Cloud Data Flow

So far so good, now let's try to run the applications with Spring Cloud Data Flow.
In this example I am using the recently released [1.3.0.M1](https://spring.io/blog/2017/08/07/spring-cloud-data-flow-1-3-0-m1-released) version which comes with a nice Angular 4 based [Dashboard UI](http://cloud.spring.io/spring-cloud-dataflow-ui/).

The runtime in Spring Cloud Data Flow is very flexible. The currently runtimes are: 
1. Cloud Foundry
2. Apache YARN
3. Kubernetes
4. Apache Mesos
5. Local Server for development

To make things simple here I am going to use the local server of the Spring Cloud Data Flow.

```bash
wget http://repo.spring.io/milestone/org/springframework/cloud/spring-cloud-dataflow-server-local/1.3.0.M1/spring-cloud-dataflow-server-local-1.3.0.M1.jar
java -jar spring-cloud-dataflow-server-local-1.3.0.M1.jar
```

After it is up and running you can access its UI at `http://localhost:9393/dashboard/#/apps`

Spring Cloud Data Flow comes with a shell which can also be used interact with the server using `stream DSL`. In this example we are going to use this one, instead of the dashboard UI.

```bash
wget http://repo.spring.io/milestone/org/springframework/cloud/spring-cloud-dataflow-shell/1.3.0.M1/spring-cloud-dataflow-shell-1.3.0.M1.jar
java -jar spring-cloud-dataflow-shell-1.3.0.M1.jar
```

Before deploying the applications make sure the applications are installed in your maven repository, since we are using here Spring Boot uber-jar hosted in maven repository. Applications packaged as a docker container is also supported. 
After the shell is up and running we need to register the applications. 

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

You can also check in the dashboard UI the result is the same:

<p><img src="/images/2017-08-22/scdf-apps.png" alt="Spring Cloud Data Flow - Apps" /></p>

Another option is to register multiple apps at once, you can define them in a properties file with keys <type>.<name>. For example in the `my-apps.properties`

```bash
stream.source-app=maven://com.example:source-app:jar:0.0.1-SNAPSHOT
stream.processor-app=maven://com.example:processor-app:jar:0.0.1-SNAPSHOT
stream.sink-app=maven://com.example:sink-app:jar:0.0.1-SNAPSHOT
```

and then you can import using:

```bash
dataflow:>app import --uri file:///<YOUR_FILE_LOCATION>/my-apps.properties
```

Next we need to define a stream.

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

And then we deploy the stream

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

If the stream is deployed successfully you should check the `sink-app` log file in order to verify that receives the processed messages. 

```bash
...
2017-08-21 17:26:52.988  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.s.c.StreamDeploymentController   : Downloading resource URI [maven://com.example:sink-app:jar:0.0.1-SNAPSHOT]
2017-08-21 17:26:54.192  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.s.c.StreamDeploymentController   : Deploying application named [sink-app] as part of stream named [myStream] with resource URI [maven://com.example:sink-app:jar:0.0.1-SNAPSHOT]
2017-08-21 17:26:54.229  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.spi.local.LocalAppDeployer       : Deploying app with deploymentId myStream.sink-app instance 0.
   Logs will be in /var/folders/9t/nr0ckt5571317_hf6b0hbty80000gn/T/spring-cloud-dataflow-5517844924337339292/myStream-1503329214194/myStream.sink-app
2017-08-21 17:26:54.237  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.s.c.StreamDeploymentController   : Downloading resource URI [maven://com.example:processor-app:jar:0.0.1-SNAPSHOT]
2017-08-21 17:26:54.950  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.s.c.StreamDeploymentController   : Deploying application named [processor-app] as part of stream named [myStream] with resource URI [maven://com.example:processor-app:jar:0.0.1-SNAPSHOT]
2017-08-21 17:26:54.962  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.spi.local.LocalAppDeployer       : Deploying app with deploymentId myStream.processor-app instance 0.
   Logs will be in /var/folders/9t/nr0ckt5571317_hf6b0hbty80000gn/T/spring-cloud-dataflow-5517844924337339292/myStream-1503329214950/myStream.processor-app
2017-08-21 17:26:54.976  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.s.c.StreamDeploymentController   : Downloading resource URI [maven://com.example:source-app:jar:0.0.1-SNAPSHOT]
2017-08-21 17:26:55.819  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.s.c.StreamDeploymentController   : Deploying application named [source-app] as part of stream named [myStream] with resource URI [maven://com.example:source-app:jar:0.0.1-SNAPSHOT]
2017-08-21 17:26:55.832  INFO 40752 --- [nio-9393-exec-2] o.s.c.d.spi.local.LocalAppDeployer       : Deploying app with deploymentId myStream.source-app instance 0.
   Logs will be in /var/folders/9t/nr0ckt5571317_hf6b0hbty80000gn/T/spring-cloud-dataflow-5517844924337339292/myStream-1503329215820/myStream.source-app
...
```

You can access the source code on my [github](http://altfatterz.github.io/spring-cloud-dafaflow-streaming-example). In a following post I will look into short lived application support using Spring Cloud Task and how this integrates again with Spring Cloud Data Flow. We might even use Kubernetes as runtime instead of Local Server, so stay tuned :)

Happy coding & learning!

