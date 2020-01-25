---
layout: post
title: Schema Evolution with Confluent Schema Registry
tags: [kafka, confluent]
---

Install the [Confluent Platform](https://docs.confluent.io/current/quickstart/index.html)

Here I chose to download the zip, then unpack it to a folder and setting the `.zshrc` file 
 
```bash
$ export CONFLUENT_HOME=<your-path>/confluent-5.3.2
$ export PATH="$CONFLUENT_HOME/bin:$PATH" 
```

Install Confluent CLI

```bash
$ curl -L https://cnfl.io/cli | sh -s -- -b $CONFLUENT_HOME/bin
```

```bash
$ confluent local status

control-center is [DOWN]
ksql-server is [DOWN]
connect is [DOWN]
kafka-rest is [DOWN]
schema-registry is [DOWN]
kafka is [DOWN]
zookeeper is [DOWN]
```

Start Kafka

```bash
$ confluent local start kafka
```

As we can see only `Kafka` and `Zookeeper` were started..

```bash
$ confluent local status

control-center is [DOWN]
ksql-server is [DOWN]
connect is [DOWN]
kafka-rest is [DOWN]
schema-registry is [DOWN]
kafka is [UP]
zookeeper is [UP]
```

Install the `Kafka Connect Datagen` from [Confluent Hub](https://www.confluent.io/hub/)

```bash
$ confluent-hub install --no-prompt confluentinc/kafka-connect-datagen:latest

Running in a "--no-prompt" mode
Implicit acceptance of the license below:
Apache License 2.0
https://www.apache.org/licenses/LICENSE-2.0
Downloading component Kafka Connect Datagen 0.2.0, provided by Confluent, Inc. from Confluent Hub and installing into /Users/zoal/apps/confluent-5.3.2/share/confluent-hub-components
Adding installation directory to plugin path in the following files:
  /Users/zoal/apps/confluent-5.3.2/etc/kafka/connect-distributed.properties
  /Users/zoal/apps/confluent-5.3.2/etc/kafka/connect-standalone.properties
  /Users/zoal/apps/confluent-5.3.2/etc/schema-registry/connect-avro-distributed.properties
  /Users/zoal/apps/confluent-5.3.2/etc/schema-registry/connect-avro-standalone.properties
```


KSQL example

Create a stream

```bash
CREATE STREAM pageviews_female AS SELECT users.userid AS userid, pageid, regionid, gender 
FROM pageviews LEFT JOIN users ON pageviews.userid = users.rowkey WHERE gender = 'FEMALE';
```

```bash
{
  "@type": "currentStatus",
  "statementText": "CREATE STREAM PAGEVIEWS_FEMALE WITH (REPLICAS = 1, PARTITIONS = 1, KAFKA_TOPIC = 'PAGEVIEWS_FEMALE') AS SELECT\n  USERS.USERID \"USERID\"\n, PAGEVIEWS.PAGEID \"PAGEID\"\n, USERS.REGIONID \"REGIONID\"\n, USERS.GENDER \"GENDER\"\nFROM PAGEVIEWS PAGEVIEWS\nLEFT OUTER JOIN USERS USERS ON ((PAGEVIEWS.USERID = USERS.ROWKEY))\nWHERE (USERS.GENDER = 'FEMALE');",
  "commandId": "stream/PAGEVIEWS_FEMALE/create",
  "commandStatus": {
    "status": "SUCCESS",
    "message": "Stream created and running"
  },
  "commandSequenceNumber": 3,
  "warnings": []
}
```


Confluent Cloud CLI

```bash
$ curl -L https://cnfl.io/ccloud-cli | sh -s -- -b $CONFLUENT_HOME/bin
```

Check the Confluent Cloud CLI version

```bash
$ ccloud version

ccloud - Confluent Cloud CLI

Version:     v0.218.0
Git Ref:     a44ec07
Build Date:  2019-12-13T01:16:19Z
Build Host:  semaphore@semaphore-vm
Go Version:  go1.12.10 (darwin/amd64)
Development: false
```

Create a new environment

```bash
$ ccloud environment create demo
```

List the environments

```bash
$ ccloud environment list

     Id    |  Name
+----------+---------+
  * t829   | default
    t25349 | demo
```

Use the new environment

```basg
$ ccloud environment use t25349
```

Create a new Kafka cluster in the GUI

TODO include an image

List the kafka clusters

```bash
$ ccloud kafka cluster list

      Id      |     Name     | Provider |    Region    | Durability | Status
+-------------+--------------+----------+--------------+------------+--------+
    lkc-9km8y | demo-cluster | gcp      | europe-west3 | LOW        | UP
```

Use the Kafka cluster:

```bash
$ ccloud kafka cluster use lkc-9km8y
```

Describe the Kafka cluster:

```bash
$ ccloud kafka cluster describe lkc-9km8y
+-------------+------------------------------------------------------------+
| Id          | lkc-9km8y                                                  |
| Name        | demo-cluster                                               |
| Ingress     |                                                        100 |
| Egress      |                                                        100 |
| Storage     |                                                       5000 |
| Provider    | gcp                                                        |
| Region      | europe-west3                                               |
| Status      | UP                                                         |
| Endpoint    | SASL_SSL://pkc-4ygn6.europe-west3.gcp.confluent.cloud:9092 |
| ApiEndpoint | https://pkac-ewz8j.europe-west3.gcp.confluent.cloud        |
+-------------+------------------------------------------------------------+
```

Spring Boot Application connected to Kafka cluster in Confluent Cloud  

https://www.confluent.io/blog/apache-kafka-spring-boot-application/


To look into:
1. ccloud-tools: https://github.com/confluentinc/ccloud-tools

