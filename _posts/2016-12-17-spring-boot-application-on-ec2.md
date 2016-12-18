---
layout: post
title: Spring Boot application on EC2
tags: [spring, aws, ec2]
---

In this blog post we are going to build a simple Spring Boot app which will expose [instance metadata](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html) information when running on an AWS EC2 instance. We are going to use [Spring Cloud AWS](https://cloud.spring.io/spring-cloud-aws/) which eases the integration with AWS.

The application [spring-boot-ec2-demo](https://github.com/altfatterz/spring-boot-ec2-demo) is packaged as a Maven artifact and is pushed to [AWS S3](https://aws.amazon.com/s3/). During launching an EC2 instance the artifact will be pulled from S3 and after little housekeeping it will be started as a service on the instance.

First we create an [IAM](https://aws.amazon.com/documentation/iam/) role which allows us push and pull content to S3 without using access and secret keys. 

<p><img src="/images/s3-admin-access.png" alt="S3 Admin Access Role" /></p>

In the picture above you can see the created IAM role with the `AmazonS3FullAccess` policy attached. 

In this example we chose `Amazon Linux AMI` with `t2.micro` instance type. When configuring the instance details we set the `IAM role` field to the previously created role as seen in the picture below.

<p><img src="/images/using-s3-admin-access.png" alt="S3 Admin Access Role" /></p>

Next in the user data section we need to provide the bootstrap script.

<p><img src="/images/aws-user-data.png" alt="S3 Admin Access Role" /></p>

With user data settings we can run a configuration script during launch. If we launch more than one instance at a time, the user data is available to all the instances in that reservation.

Our bootstrap script is the following:

```bash
#!/bin/bash
# install updates
yum update -y

# install apache httpd
yum install httpd -y

# install java 8
yum install java-1.8.0 -y
# remove java 1.7
yum remove java-1.7.0-openjdk -y

# create the working directory
mkdir /opt/spring-boot-ec2-demo
# create configuration specifying the used profile
echo "RUN_ARGS=--spring.profiles.active=ec2" > /opt/spring-boot-ec2-demo/spring-boot-ec2-demo.conf

# download the maven artifact from S3
aws s3 cp s3://spring-boot-examples/spring-boot-ec2-demo.jar /opt/spring-boot-ec2-demo/ --region=eu-central-1

# create a springboot user to run the app as a service
useradd springboot
# springboot login shell disabled
chsh -s /sbin/nologin springboot
chown springboot:springboot /opt/spring-boot-ec2-demo/spring-boot-ec2-demo.jar
chmod 500 /opt/spring-boot-ec2-demo/spring-boot-ec2-demo.jar

# create a symbolic link
ln -s /opt/spring-boot-ec2-demo/spring-boot-ec2-demo.jar /etc/init.d/spring-boot-ec2-demo

# forward port 80 to 8080
echo "<VirtualHost *:80>
  ProxyRequests Off
  ProxyPass / http://localhost:8080/
  ProxyPassReverse / http://localhost:8080/
</VirtualHost>" >> /etc/httpd/conf/httpd.conf

# start the httpd and spring-boot-ec2-demo
service httpd start
service spring-boot-ec2-demo start

# automatically start httpd and spring-boot-ec2-demo if this ec2 instance reboots
chkconfig httpd on
chkconfig spring-boot-ec2-demo on
```

And after 2 minutes we have the service up and running:

```bash
$ curl http://54.93.39.25 | jq

{
  "instanceId": "i-01abb2b9357ec2b2d",
  "publicIp": "54.93.39.25",
  "privateIp": "172.31.1.100"
} 
```

[jq](https://stedolan.github.io/jq/) is the JSONView for the command line, a must have in your toolbox. If you try it yourself change the `54.93.39.25` to the public IP of your EC2 instance.  

While creating the bootstrap script was helpful that Amazon made the `Amazon Linux` [available](https://aws.amazon.com/about-aws/whats-new/2016/11/introducing-new-amazon-linux-container-image-for-cloud-and-on-premises-workloads/) as a docker image. This way no need to worry about messing something up on the environment, just recreate the instance until you make sure the bootstrap script is working as expected.

```
$ docker run -dit amazonlinux
$ docker attach <container-id>
```

Now, let's take look how the application is exposing some instance metadata information.
 
Instance metadata is available on the instance itself via:

```
$ curl http://169.254.169.254/latest/meta-data/
```

With Spring Cloud AWS instance metadata can be directly injected into Java fields with the `@Value` annotation.

```java
@Component
@Getter
public class ApplicationInfoBean {

    @Value("${instance-id:N/A}")
    private String instanceId;

    @Value("${public-ipv4:N/A}")
    private String publicIp;

    @Value("${local-ipv4:N/A}")
    private String privateIp;
}
```

Then with a REST controller the information is exposed: 

```java
@RestController
@Slf4j
public class ApplicationInfoController {

    private final ApplicationInfoBean infoBean;

    @Autowired
    public ApplicationInfoController(ApplicationInfoBean infoBean) {
        this.infoBean = infoBean;
    }

    @GetMapping
    public ApplicationInfoBean info() {
        log.info("handling info request on instance with ip {}", infoBean.getPrivateIp());
        return infoBean;
    }
}
```

Build and copy the artifact to S3

```bash
$ mvn clean package
$ aws s3 cp target/spring-boot-ec2-demo.jar s3://spring-boot-examples/
```

Running locally:

```bash
$ java -jar target/spring-boot-ec2-demo.jar --spring.profiles.active=dev --debug
``` 

One annoying thing when running locally is that `AwsCloudEnvironmentCheckUtils.isRunningOnCloudEnvironment()` [increases the startup time](https://github.com/spring-cloud/spring-cloud-aws/issues/181) heavily and currently there is no property to override/disable this process.  

Accessing the logs of `spring-boot-ec2-demo` on the EC2 instance:

```bash
$ tail -1000f /var/log/spring-boot-ec2-demo.log
```

Accessing the logs of `httpd` on the EC2 instance:

```bash
$ sudo tail -1000f /var/log/httpd/access_log
```

In a following blog post we will look into centralized logging witch [CloudWatch Logs](http://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) service, which comes in handy when we have multiple instances of a service behind a load balancer. 
