---
layout: post
title: Deploying a Spring Boot app to AWS using Boxfuse
tags: [boxfuse, awscloud, springboot]
---

```
$ boxfuse create chucknorrisfacts -apptype=load-balanced -dbtype=postgresql
```

```
$ boxfuse info

Info about altfatterz/chucknorrisfacts in the dev environment:

App Type    : Load Balanced with Zero Downtime updates
DB Type     : PostgreSQL database
```

```
$ boxfuse fuse build/libs/chucknorrisfacts-1.0.jar
```

```
$ boxfuse ls

Images available locally:
+---------------------------------+--------------------------+-------+----------------------------+--------------+---------+---------------------+
| Image                           |         Payload          | Debug |          Runtime           |    Ports     |  Size   |    Generated at     |
+---------------------------------+--------------------------+-------+----------------------------+--------------+---------+---------------------+
| altfatterz/chucknorrisfacts:1.0 | chucknorrisfacts-1.0.jar | false | Java 8.74.02 (Spring Boot) | http -> 8080 | 67405 K | 2016-03-28 13:03:47 |
+---------------------------------+--------------------------+-------+----------------------------+--------------+---------+---------------------+
Total: 1
```

Boxfuse automatically provisions a PostgreSQL database locally, exposing the database url and credentials as environment variables
which are used by the application instances.

Here test it locally making sure the image works with VirtualBox.




```
$ boxfuse push chucknorrisfacts:1.0

Pushing altfatterz/chucknorrisfacts:1.0 ...
Verifying altfatterz/chucknorrisfacts:1.0 ...
```

Creating an AWS AMI from the image in the Boxfuse Vault.

```
$ boxfuse convert chucknorrisfacts:1.0

Waiting for AWS to create an AMI for altfatterz/chucknorrisfacts:1.0 in eu-central-1 (this may take up to 50 seconds) ...
AMI created in 01:00.529s in eu-central-1 -> ami-dc09efb3
```

Boxfuse automatically provisioned a PostgreSQL database on AWS RDS.

```
$ boxfuse run chucknorrisfacts:1.0 -env=test -ports.http=80
```




Connecting to the provisioned PostgreSQL RDS service:
```
$ boxfuse open -db -env=test
```

Removing image from the Boxfuse vault:
```
$ boxfuse rm chucknorrisfacts:1.0 -vault=true
```

It also deregisters the used AWS AMI image.


Issue with creating the load balancer with different port:
https://github.com/boxfuse/boxfuse-issues/issues/65

```
```