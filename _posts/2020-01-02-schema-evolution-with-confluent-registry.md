---
layout: post
title: Schema Evolution with Confluent Schema Registry
tags: [kafka, confluent]
---

In this post we are looking into schema evolution with [Confluent Schema Registry](https://docs.confluent.io/current/schema-registry/index.html).
 
#### Start Kafka

We are starting Kafka with `docker-compose` using the [landoop/fast-data-dev](https://hub.docker.com/r/landoop/kafka-lenses-dev) image. This will also startup other services like [Confluent Schema registry](https://www.confluent.io/confluent-schema-registry/) which we will need later on.

```yaml
version: '3'

services:
  kafka-cluster:
    image: landoop/fast-data-dev:2.2
    environment:
      ADV_HOST: 127.0.0.1         # Change to 192.168.99.100 if using Docker Toolbox
      RUNTESTS: 0                 # Disable Running tests so the cluster starts faster
      FORWARDLOGS: 0              # Disable running 5 file source connectors that bring application logs into Kafka topics
      SAMPLEDATA: 0               # Do not create sea_vessel_position_reports, nyc_yellow_taxi_trip_data, reddit_posts topics with sample Avro records.
    ports:
      - 2181:2181                 # Zookeeper
      - 3030:3030                 # Landoop UI
      - 8081-8083:8081-8083       # REST Proxy, Schema Registry, Kafka Connect ports
      - 9581-9585:9581-9585       # JMX Ports
      - 9092:9092                 # Kafka Broker
```

Before you start up the services make sure you increase the RAM for at least 4GB in your Docker for Mac/Windows settings.

To start up the services use this command:

```bash
$ docker-compose -f docker-compose/landoop-kafka-cluster.yml up
```

Open the Landoop UI in a browser `http://localhost:3030`

![lenses-ui](/images/2020-01-02/lenses-ui.png)
 
#### Avro

Primitive data types in avro:

* `null`    - No value.
* `boolean` - A binary value.
* `int`     - A 32-bit signed integer.
* `long`    - A 64-bit signed integer.
* `float`   - A single precision (32 bit) IEEE 754 floating-point number
* `double`  - A double precision (64-bit) IEEE 754 floating-point number.
* `byte`    - A sequence of 8-bit unsigned bytes.
* `string`  - A Unicode character sequence.

Complex types:

* `record`
* `enum`
* `array`
* `map`
* `union` - they are represented as JSON array, and indicate that a field may have more than one type

More information you can find here: [https://docs.oracle.com/cd/E57769_01/html/GettingStartedGuide/avroschemas.html](https://docs.oracle.com/cd/E57769_01/html/GettingStartedGuide/avroschemas.html)

#### Avro Tools

Install 

```bash
$ brew install avro-tools
```

Convert from JSON to AVRO

```bash
$ avro-tools fromjson --schema-file src/main/resources/avro/schema.avsc customer.json > customer.avro
```

A `customer.avro` should be created. 

Get the schema from `customer.avro` file

```bash
$ avro-tools getschema customer.avro
```

The schema will be printed to the standard output.

Convert from AVRO to JSON

```bash
$ avro-tools tojson --pretty customer.avro
```

#### Confluent Schema Registry

Avro schemas are stored in the [Confluent Schema Registry](https://www.confluent.io/confluent-schema-registry/). A producer sends Avro content to Kafka and the schema to Schema Registry, similarly a Consumer will ge the schema from Schema Registry will read the Avro content from Kafka.

#### Kafka Avro Console Producer and Consumer

These tools come with the Confluent Schema Registry and allow to send avro data to Kafka. The fastest way to get the binaries is using `docker run` 

```bash
$ docker run --net=host -it confluentinc/cp-schema-registry:5.3.2 /bin/bash
```

Let's start first the `kafka-avro-console-consumer` with the following command:

```bash
$ kafka-avro-console-consumer --bootstrap-server localhost:9092 --topic mytopic \
    --from-beginning \
    --property schema.registry.url=http://localhost:8081 
```

Then in another terminal start the `kafka-avro-console-producer` with the following command:

```bash
$ kafka-avro-console-producer --broker-list localhost:9092 --topic mytopic \
    --property schema.registry.url=http://localhost:8081 \
    --property value.schema='{"type":"record", "name":"Customer", "fields":[{"name":"first_name", "type":"string"}] }' 
``` 

And now we can send data:

```bash
{"first_name":"Zoltan"}
{"first_name":"Kasia"}
```

Now if we check the `kafka-avro-console-consumer` should have received the data. In the [kafka-topics-ui](http://localhost:3030/kafka-topics-ui/#/) we can see that the `mytopic` topic was created and the data is inside the topic in `avro` data type.   

![kafka-avro-console-producer](/images/2020-01-02/kafka-avro-console-producer.png)

If we send the wrong field name or wrong data type for the field value we get an error

```bash
{"name":"Zoltan"}

org.apache.kafka.common.errors.SerializationException: Error deserializing json {"name":"Zoltan"} to Avro of schema {"type":"record","name":"Customer","fields":[{"name":"first_name","type":"string"}]}
Caused by: org.apache.avro.AvroTypeException: Expected field name not found: first_name
```

#### Kafka Avro Producer and Consumer with Java




#### Schema evolution 

We get errors if we try to evolve a schema in a non-compatible way. For example let's try to register another schema to the previous `mytopic`

```bash
$ kafka-avro-console-producer --broker-list localhost:9092 --topic mytopic \
    --property schema.registry.url=http://localhost:8081 \
    --property value.schema='{"type":"string"}' 
```

As soon as we type a string value we will get the following error:

```bash
org.apache.kafka.common.errors.SerializationException: Error registering Avro schema: "string"
Caused by: io.confluent.kafka.schemaregistry.client.rest.exceptions.RestClientException: Schema being registered is incompatible with an earlier schema; error code: 409
```

Let's evolve the schema by adding another field which has a default value.

```bash
$ kafka-avro-console-producer --broker-list localhost:9092 --topic mytopic \
    --property schema.registry.url=http://localhost:8081 \
    --property value.schema='{"type":"record", "name":"Customer", "fields":[{"name":"first_name", "type":"string"}, {"name":"last_name", "type":"string", "default":"" }] }'

{"first_name":"John", "last_name":"Doe"} 
```

We should not get error this time and we see another version of the schema registered:

![schema-evolution](/images/2020-01-02/schema-evolution.png)

#### Conclusion

References
* Avro docs: https://avro.apache.org/docs/current/spec.html
* Schema Registry: https://docs.confluent.io/current/schema-registry/index.html
