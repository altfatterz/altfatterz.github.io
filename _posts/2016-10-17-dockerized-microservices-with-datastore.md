---
layout: post
title: Dockerized microservices with a datastore
tags: [docker, mysql, springboot]
---

I have created a sample [repository](https://github.com/altfatterz/dockerized-microservices) where I experiment how to setup dockerized microservices with different datastores.
In this blog post I will go through the `mysql-sample`. 

Below you can find the `Dockerfile` for the `mysql-datastore` image. Leveraging the official `mysql` image from `Docker Hub` and using couple of environment variables the mysql container is configured. `MYSQL_RANDOM_ROOT_PASSWORD` environment variable set to `yes` makes sure that a random password for the root user will be generated and printed to the standard output in the container.
The user identified with `MYSQL_USER` will be granted superuser permissions for the database specified by the `MYSQL_PASSWORD`.

```bash
FROM mysql:5.7.15
MAINTAINER altfatterz@gmail.com
ENV MYSQL_RANDOM_ROOT_PASSWORD 'yes'
ENV MYSQL_USER demo
ENV MYSQL_PASSWORD demo
ENV MYSQL_DATABASE db
```

The sample contains another `Dockerfile` which sets up a simple Spring Boot app using the previous `mysql` service. 

```bash
# The smallest Docker image with OracleJDK 8 (167MB)
FROM frolvlad/alpine-oraclejdk8:slim

# add bash and coreutils
RUN apk add --no-cache bash coreutils

MAINTAINER altfatterz@gmail.com

# We added a VOLUME pointing to "/tmp" because that is where a Spring Boot application creates working directories for
# Tomcat by default. The effect is to create a temporary file on your host under "/var/lib/docker" and link it to the
# container under "/tmp". This step is optional for the simple app that we wrote here, but can be necessary for other
# Spring Boot applications if they need to actually write in the filesystem.
VOLUME /tmp

# The project JAR file is ADDed to the container as "app.jar"
ADD mysql-sample-0.0.1-SNAPSHOT.jar app.jar

# script files related to check if the datastore is “ready”
ADD wait-for-it.sh wait-for-it.sh
ADD start.sh start.sh

EXPOSE 8080

# You can use a RUN command to "touch" the jar file so that it has a file modification time
# (Docker creates all container files in an "unmodified" state by default)
# This actually isn’t important for the simple app that we wrote, but any static content (e.g. "index.html")
# would require the file to have a modification time.
RUN bash -c 'touch /app.jar'

CMD ["./start.sh"]
```

I use the [wait-for-it](https://github.com/vishnubob/wait-for-it) bash script to test and wait for the availability of the TCP host and port where the `mysql` service will be available.

```bash
#!/bin/bash

# wait for 15 seconds until mysql is up
./wait-for-it.sh -t 15 mysql:3306

if [ $? -eq 0 ]
then
  # To reduce Tomcat startup time we added a system property pointing to "/dev/urandom" as a source of entropy.
  java -Djava.security.egd=file:/dev/./urandom -jar app.jar
fi
```

With the help of [docker-mave-plugin](https://github.com/spotify/docker-maven-plugin) the `altfatterz/mysql-sample` and `altfatterz/mysql` images are generated during maven `package` phase.   

```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>${docker.maven.plugin.version}</version>
    <executions>
        <execution>
            <id>mysql</id>
            <!-- bind docker commands to maven phases -->
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
            <configuration>
                <imageName>${docker.image.prefix}/mysql</imageName>
                <!-- the contents of dockerDirectory will be copied into ${project.build.directory}/docker folder -->
                <dockerDirectory>${project.basedir}/src/main/db</dockerDirectory>
            </configuration>
        </execution>
        <execution>
            <id>web</id>
            <!-- bind docker commands to maven phases -->
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
            <configuration>
                <imageName>${docker.image.prefix}/${project.artifactId}</imageName>
                <dockerDirectory>${project.basedir}/src/main/docker</dockerDirectory>
                <!-- Use the resources element to copy additional files, such as the service's jar file -->
                <resources>
                    <resource>
                        <targetPath>/</targetPath>
                        <directory>${project.build.directory}</directory>
                        <include>${project.build.finalName}.jar</include>
                    </resource>
                </resources>
            </configuration>
        </execution>
    </executions>
</plugin>
```

And finally with `Docker Compose` I can start both services with a single command. Note here that the `depends_on` only guarantees the order of the service startup, in this case `mysql-datastore` will be started first. `Docker Compose` does not wait until a container is "ready", that is why I needed to use the `wait-for-it` wrapper script.     

```bash
version: '2'
services:
  mysql:
    container_name: mysql-datastore
    image: altfatterz/mysql
    # expose 3306 port to host, handy to inspect the database from the host machine
    ports:
      - "3306:3306"
  web:
    container_name: service-using-mysql-datastore
    image: altfatterz/mysql-sample
    # docker-compose will start services in dependency order. Here mysql service will be started before web
    # depends_on will not wait for mysql to be "ready" before starting "web". If you need a service to be ready see
    # https://docs.docker.com/compose/startup-order/
    depends_on:
      - mysql
    # open ports for tomcat and remote debugging
    ports:
      - "8080:8080"
      - "8000:8000"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql/db?autoReconnect=true&useSSL=false
      SPRING_DATASOURCE_USERNAME: demo
      SPRING_DATASOURCE_PASSWORD: demo
```

There are other samples using Riak and MongoDB in this [repository](https://github.com/altfatterz/dockerized-microservices) and I am planning to add more in the future.     