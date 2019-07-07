---
layout: post
title: Spring Boot app on OpenShift
tags: [springboot, openshift, kubernetes]
---

In this blogpost we are showing how to deploy Spring Boot application to OpenShift levaraging the [fabric8-maven-plugin](https://github.com/fabric8io/fabric8-maven-plugin) and [Spring Cloud Kubernetes](https://spring.io/projects/spring-cloud-kubernetes)

### Minishift setup

First we need an OpenShift cluster. On Mac is very easy to get started:

```bash
$ brew cask install minishift
$ brew cask install virtualbox
```

By default, Minishift will use the relevant hypervisor based on the host operating system, 
here we instruct to use `VirtualBox` instead of `xhyve` on OSX.

```bash
$ minishift start --vm-driver=virtualbox
```

We can verify that our single node OpenShift cluster is running using:

```bash
$ minishift status

Minishift:  Running
Profile:    minishift
OpenShift:  Running (openshift v3.11.0+3a34b96-217)
DiskUsage:  57% of 19G (Mounted On: /mnt/sda1)
CacheUsage: 1.677 GB (used by oc binary, ISO or cached images)
```

and get the cluster IP using:


```bash
$ minishift ip

192.168.99.101
```

At the time of writing we were using the Minishift version:

```bash
$ minishift version

minishift v1.33.0+ba29431
```

### OpenShift Client

Next, we need to install the `OpenShift Client` in order to interact with our OpenShift cluster from the command line:  

```bash
$ brew install openshift-cli
```

```bash
$ oc version

Client Version: version.Info{Major:"4", Minor:"1+", GitVersion:"v4.1.0+b4261e0", GitCommit:"b4261e07ed", GitTreeState:"clean", BuildDate:"2019-05-18T05:40:34Z", GoVersion:"go1.12.5", Compiler:"gc", Platform:"darwin/amd64"}
Server Version: version.Info{Major:"1", Minor:"11+", GitVersion:"v1.11.0+d4cacc0", GitCommit:"d4cacc0", GitTreeState:"clean", BuildDate:"2019-07-05T15:07:29Z", GoVersion:"go1.10.8", Compiler:"gc", Platform:"linux/amd64"}
```

Next we login with `developer/developer`

```bash
$ oc login

Authentication required for https://192.168.99.101:8443 (openshift)
Username: developer
Password:
```

### OpenShift project

We create a new project called `boot`

```bash
$ oc new-project boot

Now using project "boot" on server "https://192.168.99.101:8443".
```

```bash
$ oc status

In project boot on server https://192.168.99.101:8443

You have no services, deployment configs, or build configs.
Run 'oc new-app' to create an application.
```

### PostgreSQL service 

For the example we will need a `PostgreSQL` service. In order to provision this service we are using the `OpenShift Web Console` provided at

```bash
https://<minishift-ip>:8443/console/
``` 

After logging in as `developer`/`developer` we select the project `boot` created in the previous step.
We make sure that the `boot` project is be visible in the top left corner, otherwise when creating resources we might create them in another project.  
On the left we select the `catalog`, then `Databases` and then `Postgres`.
Select `PostgreSQL` click `Next`, then `Create`, leaving everything at default. This will create a `PostgreSQL` database.

```bash
The following service(s) have been created in your project: boot.

       Username: userT0G
       Password: MMkglTP6tRe5mJL0
  Database Name: sampledb
 Connection URL: postgresql://postgresql:5432/
``` 

If we look around in the `OpenShift Web Console` we will see that many things were created: a `deployment`, with one pod running, a `persistent volume claim`, a `secret`. 

In the terminal we can also view the created service using:

```bash
$ oc get service

NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
postgresql   ClusterIP   172.30.1.92   <none>        5432/TCP   4m
```

Note, that with the following command we can always verify in which project we are:

```bash
oc project

Using project "boot" on server "https://192.168.99.101:8443".
```

Lets connect to recently created `postgresql` service and create a `customers` table

```bash
$ oc get pods

NAME                 READY   STATUS    RESTARTS   AGE
postgresql-1-zzrlp   1/1     Running   0          30m
```
 
```bash
$ oc rsh postgresql-1-zzrlp

sh-4.2$ psql sampledb userT0G

psql (9.6.10)
Type "help" for help.

sampledb=
``` 

```bash
sampledb=# create table customers (id BIGSERIAL,  first_name varchar(255) not null, last_name varchar(255) not null,  PRIMARY KEY (id));
CREATE TABLE
sampledb=# insert into customers values (1, 'Sam', 'Sung');
INSERT 0 1
sampledb=# \d
                List of relations
 Schema |       Name       |   Type   |  Owner
--------+------------------+----------+----------
 public | customers        | table    | postgres
 public | customers_id_seq | sequence | postgres
(2 rows)
```

In order to disconnect from the `postgresql` and `container` use:
 
```bash
sampledb-# \q
sh-4.2$ exit
exit
command terminated with exit code 127
```

### Demo application

Get the bits of the demo application found at `https://github.com/altfatterz/openshift-spring-boot-demo`

```bash
$ git clone https://github.com/altfatterz/openshift-spring-boot-demo
```

The demo contains two Spring Boot applications `customer-service` and `order-service`.  

Let's build the applications from the `openshift-spring-boot-demo` folder

```bash
$ mvn clean install
```

In the logs we will see that is using the [`fabric8-maven-plugin`](https://github.com/fabric8io/fabric8-maven-plugin) to generate the openshift resource descriptors (via `fabric8:resource` goal) and docker images (via `fabric8:build`) for us.

```xml
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>fabric8-maven-plugin</artifactId>
    <version>${fabric8.maven.plugin.version}</version>
    <executions>
        <execution>
            <id>fmp</id>
            <goals>
                <goal>resource</goal>
                <goal>build</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

The `fabric8:build` goal is using by default the `Openshift` mode with `S2I` (source-to-image) strategy.

```bash
[INFO] F8: Running in OpenShift mode
[INFO] F8: Using OpenShift build with strategy S2I
```
 
The generated image is based on CentOS and supports Java 8 and 11. https://hub.docker.com/r/fabric8/s2i-java

The `Source-to-Image` means that a build is done by the OpensShift cluster. We can check the created pods which were created while running the builds.

```bash
$ oc get pods

customer-service-s2i-1-build   0/1     Completed   0          2m
order-service-s2i-1-build      0/1     Completed   0          1m
...
```

The created docker images are stored in OpenShift’s integrated Docker Registry:

```bash
$ oc get imagestream

NAME               DOCKER REPO                             TAGS     UPDATED
customer-service   172.30.1.1:5000/boot/customer-service   latest   14 minutes ago
order-service      172.30.1.1:5000/boot/order-service      latest   14 minutes ago
```
                                        
More information about `Builds and Image Streams` can be found [here](https://docs.openshift.com/enterprise/3.0/architecture/core_concepts/builds_and_image_streams.html)                                        
                                        
We can start the `customer-service` application locally using:

```bash
$ java -jar customer-service/target/customer-service-0.0.1-SNAPSHOT.jar --spring.profiles.active=local
```

With the `local` Spring profile is using an embedded H2 database. Let's access the `/customers` endpoint:

```bash
$ http :8081/customers

HTTP/1.1 401
```

The endpoint requires a JWT token, where the JWT token secret can be externally configured.
After creating a JWT token (see `JwtTokenGenerator`) and sending it as an `Authorization` header with get a 200 response:  
 
```bash
$ export HEADER='Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huZG9lIiwiYXV0aG9yaXRpZXMiOiJ1c2VyIn0.wUZ3aXYwmn4RoW1Tvpwq6x_AsAm4PdcD6U417SgzFfg'
$ http :8081/customers $HEADER

HTTP/1.1 200
[
    {
        "firstName": "John",
        "id": 1,
        "lastName": "Doe"
    },
    {
        "firstName": "Jane",
        "id": 2,
        "lastName": "Doe"
    }
]

``` 

### ConfigMap and Secret

Before deploying the `customer-service` application let's create a `ConfigMap` and `Secret` with keys that the `customer-service` is using:  

```bash
oc create configmap openshift-spring-boot-demo-config-map \ 
    --from-literal=greeting.message="What is OpenShift?"
    
oc create secret generic openshift-spring-boot-demo-secret \
    --from-literal=greeting.secret="H3ll0" \
    --from-literal=jwt.secret="demo" \
    --from-literal=spring.datasource.username="userT0G" \
    --from-literal=spring.datasource.password="MMkglTP6tRe5mJL0"
```

To view the created resources we can use:

```bash
oc get configmap openshift-spring-boot-demo-config-map -o yaml
oc get secret openshift-spring-boot-demo-secret -o yaml
```

### Deploy to OpenShift

To deploy we use the following command in the `customer-service` folder

```bash
$ mvn fabric8:deploy
```

A `route` will be created for the `customer-service`

```bash
$ oc get route 

NAME               HOST/PORT                                     PATH   SERVICES           PORT   TERMINATION   WILDCARD
customer-service   customer-service-boot.192.168.99.101.nip.io          customer-service   8080                 None
```

We can see that the JWT token secret is found from the OpenShift Secret and that the application connects successfully to the `postgresql` service.

```bash
$ http  customer-service-boot.192.168.99.101.nip.io/customers $HEADER

HTTP/1.1 200
[
    {
        "id": 1,
        "firstName": "Sam",
        "lastName": "Sung"
    }
]
```

For more information about the deployment we can use:

```bash
$ oc describe dc customer-service

Name:		customer-service
Namespace:	boot
Created:	23 minutes ago
Labels:		app=customer-service
		group=com.example
		provider=fabric8
		version=0.0.1-SNAPSHOT
Annotations:	fabric8.io/git-branch=master
		fabric8.io/git-commit=98a8f456aab264ffd7cdf6b6e59779522f694828
		fabric8.io/scm-tag=HEAD
		fabric8.io/scm-url=https://github.com/spring-projects/spring-boot/spring-boot-starter-parent/customer-service
Latest Version:	1
Selector:	app=customer-service,group=com.example,provider=fabric8
Replicas:	1
Triggers:	Config, Image(customer-service@latest, auto=true)
Strategy:	Rolling
Template:
Pod Template:
  Labels:	app=customer-service
		group=com.example
		provider=fabric8
		version=0.0.1-SNAPSHOT
  Annotations:	fabric8.io/git-branch: master
		fabric8.io/git-commit: 98a8f456aab264ffd7cdf6b6e59779522f694828
		fabric8.io/scm-tag: HEAD
		fabric8.io/scm-url: https://github.com/spring-projects/spring-boot/spring-boot-starter-parent/customer-service
  Containers:
   spring-boot:
    Image:	172.30.1.1:5000/boot/customer-service@sha256:872969674a0af4e8973ae5206b8c4b854247d1319bc5638371fb5b09f7261b51
    Ports:	8080/TCP, 9779/TCP, 8778/TCP
    Host Ports:	0/TCP, 0/TCP, 0/TCP
    Liveness:	http-get http://:8080/actuator/health delay=180s timeout=1s period=10s #success=1 #failure=3
    Readiness:	http-get http://:8080/actuator/health delay=10s timeout=1s period=10s #success=1 #failure=3
    Environment:
      GREETING_MESSAGE:			<set to the key 'greeting.message' of config map 'openshift-spring-boot-demo-config-map'>	Optional: false
      GREETING_SECRET:			<set to the key 'greeting.secret' in secret 'openshift-spring-boot-demo-secret'>		Optional: false
      JWT_SECRET:			<set to the key 'jwt.secret' in secret 'openshift-spring-boot-demo-secret'>			Optional: false
      SPRING_DATASOURCE_USERNAME:	<set to the key 'spring.datasource.username' in secret 'openshift-spring-boot-demo-secret'>	Optional: false
      SPRING_DATASOURCE_PASSWORD:	<set to the key 'spring.datasource.password' in secret 'openshift-spring-boot-demo-secret'>	Optional: false
      KUBERNETES_NAMESPACE:		 (v1:metadata.namespace)
    Mounts:				<none>
  Volumes:				<none>

Deployment #1 (latest):
	Name:		customer-service-1
	Created:	23 minutes ago
	Status:		Complete
	Replicas:	1 current / 1 desired
	Selector:	app=customer-service,deployment=customer-service-1,deploymentconfig=customer-service,group=com.example,provider=fabric8
	Labels:		app=customer-service,group=com.example,openshift.io/deployment-config.name=customer-service,provider=fabric8,version=0.0.1-SNAPSHOT
	Pods Status:	1 Running / 0 Waiting / 0 Succeeded / 0 Failed

Events:
  Type		Reason			Age	From				Message
  ----		------			----	----				-------
  Normal	DeploymentCreated	23m	deploymentconfig-controller	Created new replication controller "customer-service-1" for version 1
```

The above worked because we provided a YAML fragment (`deployment.yml`) in the `/src/main/fabric8` folder which the `Fabric8 Maven plugin` used when creating the deployment configuration.
The plugin is also adding `liveness` and `readiness` probes for the correct `/actuator/health` endpoint.

```yml
spec:
  template:
    spec:
      containers:
      - env:
        - name: SPRING_DATASOURCE_USERNAME
          valueFrom:
            secretKeyRef:
              name: openshift-spring-boot-demo-secret
              key: spring.datasource.username
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: openshift-spring-boot-demo-secret
              key: spring.datasource.password
```

We are leveraging the `Spring Cloud Kubernetes` project which defaults to `kubernetes` profile when running on `OpenShift` cluster, so in our `application-kubernetes.yml` we configured:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgresql:5432/sampledb
```

The `username` and `password` for the `postgresql` service is resolved from the `openshift-spring-boot-demo-secret` secret.

### ConfigMap reload

After deploying the `order-service` application as well with `mvn fabric8:deploy` we will see two routes:

```bash
$ oc get routes

NAME               HOST/PORT                                     PATH   SERVICES           PORT   TERMINATION   WILDCARD
customer-service   customer-service-boot.192.168.99.101.nip.io          customer-service   8080                 None
order-service      order-service-boot.192.168.99.101.nip.io             order-service      8080                 None 
```

Both services resolve the `greeting.message` from the `openshift-spring-boot-demo-config-map` ConfigMap as configued in the `/src/main/fabric8/deloyment.yml` YAML fragment 

```bash
$ http customer-service-boot.192.168.99.101.nip.io/message

What is OpenShift?
```

```bash
$ http order-service-boot.192.168.99.101.nip.io/message

What is OpenShift?
```

`Spring Cloud Kubernetes` enables us that when a ConfigMap value is changed then application is reloaded. There are different levels of reload: `refresh`, `restart_context` and `shutdown`. In this example we are using `restart_context` where the Spring ApplicationContext is gracefully restarted and the beans are created with the new configuration.

```yaml
spring:
  cloud.kubernetes.reload:
    enabled: true
    strategy: restart_context
    

management:
  endpoint:
    restart:
      enabled: true    
```

In order this two work we still need to configure the common `openshift-spring-boot-demo-config-map` ConfigMap as a value of the `spring.cloud.kubernetes.config.name` key in the  `bootstrap.yml`. 

The easiest way to change the Config Map is with `OpenShift Web Console` under `Resources` > `Config Maps`. Select `openshift-spring-boot-demo-config-map` then `Actions > Edit`. After changing the `greeting.message` message to `What is Kubernetes?` both of our applications are restarted and then they serve the new value:   

```bash
$ http customer-service-boot.192.168.99.101.nip.io/message

What is Kubernetes?
```

```bash
$ http order-service-boot.192.168.99.101.nip.io/message

What is Kubernetes?
```

### Conclusion

In this blogpost, we

- installed Minishift and OpenShift Console
- provisioned a PostgreSQL service
- deployed a Spring Boot application to Minishift which used a provisioned PostgreSQL service
- configured the JWT token secret as an OpenShift Secret
- shared a ConfigMap between two different Spring Boot services with reload enabled

You can find the source code here: [https://github.com/altfatterz/openshift-spring-boot-demo](https://github.com/altfatterz/openshift-spring-boot-demo)