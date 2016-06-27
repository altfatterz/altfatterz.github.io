---
layout: post
title: Centralized configuration with Spring Cloud Config
tags: [spring]
---

In this blog post I am looking into Spring Cloud Config project which is a great tool for centralized configuration management in a microservice environment.
I have prepared a simple example project which you can find on my [github](git clone https://github.com/altfatterz/spring-cloud-config-example) account. It contains a `config-service` leveraging Spring Cloud Config Server and three little services using Spring Cloud Config Client.

Clone the configuration and example repositories and build the project into your home directory.

```sh
$ git clone https://github.com/altfatterz/spring-cloud-config-example-repo
$ git clone https://github.com/altfatterz/spring-cloud-config-example
$ cd spring-cloud-config-example
$ mvn clean install
```

Before starting the `config-service` you need to start `rabbitmq` in another terminal. More on this later why it is needed for this example.

```sh
java -Dsecurity.user.name=config \
     -Dsecurity.user.password=verysecure \
     -Dspring.cloud.config.server.git.uri=${HOME}/spring-cloud-config-example-repo \
     -Dencrypt.key=foobarbaz \
     -jar config-service/target/config-service-0.0.1-SNAPSHOT.jar
```

The `config-service` is protected with HTTP Basic security and the credentials are set using the system properties. Using the `spring.cloud.config.server.git.uri` we tell where it can find the git repository with the configurations
With the `encrypt.key` we set a symmetric key which is used for decryption of encrypted content before sending it to clients.

After the `config-service` started successfully request the following resource from the `config-service`:

```sh
$ curl http://config:verysecure@localhost:8888/foo.service/default
```

The resource follows the following format `/{application}/{profile}[/{label}]`. The `label` defaults to `master`. The request is handled by `ResourceController` from Spring Cloud Config Server, which was added using the `@EnableConfigServer` annotation.

It will return the following response:

```json
{
    "name": "foo-service",
    "profiles": [
        "default"
    ],
    "label": null,
    "version": "aec3d698ed6d4d4d006a6ffe95a775340a05c0c7",
    "propertySources": [
        {
            "name": "https://github.com/altfatterz/spring-cloud-config-example-repo/foo-service.properties",
            "source": {
                "info.app.name": "Foo Service",
                "secret-message": "a702d2b5b0c6bc2db67e7d487c6142e7c23254108503d1856ff516d0a64bbd3663a2514a86647dcf8467d042abcb8a6e",
                "server.port": "8081"
            }
        },
        {
            "name": "https://github.com/altfatterz/spring-cloud-config-example-repo/application.properties",
            "source": {
                "message": "Ria, Ria, Hungaria!"
            }
        }
    ]
}
```

As you can see from the response the `foo-service` should start on the 8081 port and has a `message` property as well as a `secret-message` property.
Next, start up the `foo-service` using the command:

```sh
$ java -Dspring.cloud.config.uri=http://config:verysecure@localhost:8888 \
       -jar foo-service/target/foo-service-0.0.1-SNAPSHOT.jar
```

With the `spring.cloud.config.uri` we tell the `foo-service` where it can find the `config-service` to get the his configuration.
After the `foo-service` started successfully we can check the visible properties for the `for-service` via the `/env` Spring Boot Actuator endpoint.

```sh
$ http://localhost:8081/env

{
    ...
    server.ports: {
        local.server.port: 8081
    },
    configService:https://github.com/altfatterz/spring-cloud-config-example-repo/foo-service.properties: {
        server.port: "8081",
        info.app.name: "Foo Service",
        secret-message: "Spring Cloud Rocks!"
    }
    ...
}
```

As you can see in the response the `secret-message` was decrypted.
Spring Cloud Config Client has a nice feature that it can re-initialize Spring beans when configuration changes using the `@RefreshScope` annotation.
In oder to try it let's change the `message` property in the `application.properties` in your local `spring-cloud-config-example-repo` repository and commit the changes.
You will see that the `config-service` already sees the change when accessing the `http://localhost:8888/foo.service/default`, however the `foo-service` does not.
In order to signal a to `foo-service` to re-initialize the beans annotated with `@RefreshScope` you need to send an empty body request to `/refresh` endpoint. Note also that the re-initialization occurs lazily when they are used and not on the moment of sending the `/refresh` request.

```sh
$ curl -d{} http://localhost:8081/refresh
```








```
$ ngrok http 8888
```

```
$ rabbitmq-server
```

```
$ git clone https://github.com/altfatterz/spring-cloud-config-example
$ cd spring-cloud-config-example
$ mvn clean install
$ java -jar /config-service
$ java -jar
```


