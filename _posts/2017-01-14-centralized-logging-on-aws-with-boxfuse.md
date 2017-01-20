---
layout: post
title: Centralized logging on AWS with Boxfuse
tags: [aws, cloudwatch, spring, boxfuse]
---

In a [previous]() post we saw what steps we needed to do in order to configure a service running on multiple EC2 instances to use CloudWatch Logs as a centralized logging tool.
In this post we will use [Boxfuse]() to deploy our [cloudwatch-logs-demo]() app and configure logging leveraging again CloudWatch Logs.
  
```bash
$ git clone https://github.com/altfatter/boxfuse-demo
$ cd boxfuse-demo
$ mvn clean package
```

```bash
$ java -jar target/java -jar target/boxfuse-demo-0.0.1-SNAPSHOT.jar
```

### Create a Boxfuse application

First we create a new Boxfuse application in [Boxfuse console](https://console.boxfuse.com/) with the following command. 

```bash
$ boxfuse create -app.type=load-balanced -logs.type=cloudwatch-logs

Creating cloudwatch-logs-demo ...
Mapping cloudwatchlogsdemo-dev-altfatterz.boxfuse.io to 127.0.0.1 ...
Created app cloudwatch-logs-demo (type: load-balanced, db: none, logs: cloudwatch-logs)
```

In `Boxfuse Console` you should see the following:

<p><img src="/images/boxfuse-console.png" alt="Log Group" /></p>

The details regarding the type of application you can find [here](https://boxfuse.com/docs/apptypes). The `load-balanced` type makes sure our application will be setup with an elastic load balancer with an auto-scaling group. 
Note that if you run `boxfuse run` before it will automatically create the application in Boxfuse console with the default application type which is `Single Instance (Elastic IP)`. In order to fix it, you could destroy the application with `boxfuse destroy` and then re-create the application with the `boxfuse create` command.       
 
### Running locally with on VirtualBox

#### Create a Boxfuse Image:

```bash
$ boxfuse fuse

Fusing Image for cloudwatch-logs-demo-0.0.1.jar (Spring Boot) ...
Image fused in 00:04.964s (69709 K) -> altfatterz/cloudwatch-logs-demo:0.0.1
```

#### List available images locally:

```bash
$ boxfuse ls 

Images available locally:
+---------------------------------------+--------------------------------+-------+-----------------------------+--------------+---------+---------------------+
| Image                                 |            Payload             | Debug |           Runtime           |    Ports     |  Size   |    Generated at     |
+---------------------------------------+--------------------------------+-------+-----------------------------+--------------+---------+---------------------+
| altfatterz/cloudwatch-logs-demo:0.0.1 | cloudwatch-logs-demo-0.0.1.jar | false | Java 8.112.16 (Spring Boot) | http -> 8080 | 69709 K | 2017-01-14 19:14:33 |
+---------------------------------------+--------------------------------+-------+-----------------------------+--------------+---------+---------------------+
```

#### Launch an instance:

```bash
$ boxfuse run altfatterz/cloudwatch-logs-demo:0.0.1 '-envvars.SPRING_PROFILES_ACTIVE=$BOXFUSE_ENV'

Launching Instance of altfatterz/cloudwatch-logs-demo:0.0.1 on VirtualBox ...
Forwarding http port localhost:8080 -> vb-13f74e0b:8080
Instance launched in 00:02.894s -> vb-13f74e0b
Waiting for payload to start on vb-13f74e0b:8080 (expecting HTTP 200 at /health within 300s)
...
vb-13f74e0b =>  2017-01-14 18:16:31.448  INFO 667 --- [           main] c.example.CloudWatchLogsDemoApplication  : The following profiles are active: boxfuse
...
Successfully started payload in 00:29.097s -> http://127.0.0.1:8080
```

The `run` command auto-detects the port configured in the `server.port` property in Spring Boot' configuration. You can override the port with the `-port.http=<port>` option.  If the specified port is already taken, it will select another port. It is worth to note, that that the `boxfuse run` command would have fused the image for us if there is no boxfuse image yet, we just wanted to show the steps what is happening.
By default the `boxfuse` profile is activated, with the `-envvars.SPRING_PROFILES_ACTIVE=$BOXFUSE_ENV` we specify a profile which corresponds to the current [Boxfuse environment](https://boxfuse.com/docs/environments) (in this case `dev`). Note the `BOXFUSE_ENV` environment variable is only available within the Boxfuse instance, so you must prevent variable expansion by using single quotes.        
 
#### Viewing instances:

```bash
$ boxfuse ps

Running Instances on VirtualBox in the dev environment :
+-------------+---------------------------------------+---------------------+-----------------------+---------------------+
|  Instance   |                 Image                 |        Type         |          URL          |     Launched at     |
+-------------+---------------------------------------+---------------------+-----------------------+---------------------+
| vb-13f74e0b | altfatterz/cloudwatch-logs-demo:0.0.1 | 4 CPU / 1024 MB RAM | http://127.0.0.1:8080 | 2017-01-14 19:16:22 |
```

You can shut down the instance with `boxfuse kill <instance-id>` command. Images can be removed with the `boxfuse rm <image>` command, which implicitly shuts down running all of its instances before deleting the image.  


### Run on AWS 

#### Pushing image to Boxfuse Vault

```bash
$ boxfuse push  altfatterz/cloudwatch-logs-demo:0.0.1
```

#### Configure security group and IAM profile

```bash
$ boxfuse cfg cloudwatch-logs-demo -env=test -securitygroup=sg-cdc7cca5 -instanceprofile=arn:aws:iam::617057429158:instance-profile/CloudWatch-Logs-Write-Access
```

#### Launch instances

The following command 

```bash
$ boxfuse run altfatterz/cloudwatch-logs-demo:0.0.1 '-envvars.SPRING_PROFILES_ACTIVE=$BOXFUSE_ENV' -env=test -capacity=2:t2.micro
```
Boxfuse will create an Amazon AMI from the image `altfatterz/cloudwatch-logs-demo:0.0.1` stored in Boxfuse Vault. 

#### Access the service

To access the service you can run the following command which will launch the url in your default browser:

```bash
$ boxfuse open -env=test
```
 
#### CloudWatch Logs 
 
Boxfuse creates a `boxfuse\<boxfuse-environment>` log group with `<username>/<application-name>` log stream with a json structured log entry: 

```json
{
    "instance": "i-0f99ce9d5ed52ea27",
    "image": "altfatterz/cloudwatch-logs-demo:0.0.1",
    "level": "INFO",
    "message": "2017-01-15 14:39:44.322  INFO 907 --- [nio-8080-exec-5] com.example.ApplicationInfoController    : handling info request on instance with ip 172.31.30.177"
}
``` 

Log entries from CloudWatch Logs can be streamed via the command: 

```bash
$ boxfuse logs -logs.tail -env=test
```


look at: 
https://boxfuse.com/blog/cloudwatch-logs
https://boxfuse.com/vs/docker


https://github.com/boxfuse/boxfuse-issues/issues/117

Starting with Boxfuse Client 1.24.0.1207 new applications are automatically configured for CloudWatch Logs by default, both when using the client as well as the console.