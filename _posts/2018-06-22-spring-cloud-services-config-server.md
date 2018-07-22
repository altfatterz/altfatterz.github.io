---
layout: post
title: Spring Cloud Services - Config Server
tags: [springcloudservices, springcloud, spring]
---

Spring Cloud Config is a great tool to externalize configuration in a microservice architecture. It has server and client side support. In this blog post we are going to look into how to set up the server part locally and on Cloud Foundry and use it with a client application.

### Config Server running locally

The config server can be embedded in a Spring Boot application using the `@EnableConfigServer` annotation, but here we look into how to run it using the [Spring Boot CLI](https://cloud.spring.io/spring-cloud-cli/) with [Spring Cloud CLI](https://cloud.spring.io/spring-cloud-cli/) extension installed.  
The `spring cloud --list` lists the available services which can be started

```bash
spring cloud --list
configserver dataflow eureka h2 hystrixdashboard kafka zipkin
``` 

By default running `spring cloud configserver` it will run in the "native" profile and serving configuration from the local directory ./launcher
We can customise the config server with a `configserver.yml` file inside the `~/.spring-cloud` folder

```yaml
spring:
  profiles:
    active: git

  cloud:
    config:
      server:
        git:
          uri: file://${user.home}/projects/cloudfoundry/config-repo
```

Inside the `config-repo` we create an example `foo-service.yml` (`foo-service` will be our client application) file with the content

```yaml
foo:
  message: Hallo
  secret: '{cipher}6050a8af7826c89dcd30c9046f484a5a169d27beb40a67db72440474c5cc6d77'
```

The `foo.secret` value was encrypted using 

```bash
spring encrypt Gruetzi --key s3cr3t
```

We set the `ENCRYPT_KEY` environment variable before starting the `config server` 

```bash
export ENCRYPT_KEY=s3cr3t
``` 

And we start up the `config server` using

```bash
spring cloud configserver
```

We can easily verify that the `config server` knows about the external configuration of the `foo-service` 

```bash
http :8888/foo-service/default
```

```json
{
    "label": null,
    "name": "foo-service",
    "profiles": [
        "default"
    ],
    "propertySources": [
        {
            "name": "file:///Users/zoal/projects/cloudfoundry/config-repo/foo-service.yml",
            "source": {
                "foo.message": "Hallo",
                "foo.secret": "6050a8af7826c89dcd30c9046f484a5a169d27beb40a67db72440474c5cc6d77"
            }
        }
    ],
    "state": null,
    "version": null
}
```

After starting the `foo-service` application we can see in the logs:

```
[           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at : http://localhost:8888
```

The `foo-service` is using `config-first bootstrap` and with the `spring.cloud.config.uri` locates the config server which is set to `http://localhost:8888` by default.

With a simple controller we print out the `foo.message` and `foo.secret` properties which we can verify with

```
http :8080 -a user:password
Hallo/Gruetzi
```

### Config Server on Pivotal Cloud Foundry (PCF)

So far so good. Let's setup this little example on a `Pivotal Web Services` (an instance of PCF hosted by Pivotal) where we will use the [Config Server](http://docs.pivotal.io/spring-cloud-services/1-5/common/config-server/index.html) from [Spring Cloud Service](http://docs.pivotal.io/spring-cloud-services/1-5/common/index.html).

First we need to install the [CloudFoundry CLI](https://github.com/cloudfoundry/cli) and create a [Pivotal Web Services](https://run.pivotal.io/) account.

After we pushed the [config-rep](https://github.com/altfatterz/config-repo) to GitHub we can configure a `config-server` service instance with this shared git repository. 
We will use the `p-config-server` service with the `trial` plan, which we found from that is available when as a response to `cf marketplace`
We create a service instance named `config-server` with the following command

```bash
cf cs p-config-server trial config-server -c config-server-setup.json
```  

where the `config-server-setup.json` contains the shared git repository: 

```json
{
  "git": {
    "uri": "https://github.com/altfatterz/config-repo.git"
  }
}
```

We wait until the service instance is created, the status can be easily checked using `cf service config-server`.

Next we need to set the encrypt key for the `config-server`. By default the `config-server` will decrypt the secrets and return the decrypted secrets to the client applications like in our case the to the `foo-service`. It is important that we protect the `config server` and have https connection between the `config server` and its client applications.
It is also possible that the client applications decrypt the secrets locally. In this case we need to set the `spring.cloud.config.server.encrypt.enabled=false` in the `bootstrap.yml` and set the encrypt key for the client applications.         

In `Pivotal Web Services` from the `Config Server Dashboard` we can get the `config-server` service instance configuration and by adding the `"encrypt":{"key":"s3cr3t"}` we can update the service instance:

```bash
cf update-service config-server -c '{"count":1,"git":{"uri":"https://github.com/altfatterz/config-repo.git"}, "encrypt":{"key":"s3cr3t"} }'
```

Then we have to add the following dependency to the `foo-service`: 
  
```xml
<dependency>
    <groupId>io.pivotal.spring.cloud</groupId>
    <artifactId>spring-cloud-services-starter-config-client</artifactId>
</dependency>
```

edit the `bootstrap.yml` configuration file 

```yaml
spring:
  cloud:
    config:
      uri: ${vcap.services.config-server.credentials.uri:http://localhost:8888}
```

and add the `mainfest.yml` file

```yaml
applications:
- name: foo-service
  host: foo-service
  memory: 756M
  instances: 1
  path: target/foo-service-0.0.1-SNAPSHOT.jar
  services:
  - config-server
```

and deploy the service using `cf push` from the `foo-service` directory.

After we verify that the app is up and running via `cf service foo-service` we can test our service using

```bash
http foo-service.cfapps.io -a user:password

Hallo/Gruetzi
```

Let's check how does this work. When the `foo-service` was bound the to  `config-server` service instance the connection details were set in the `VCAP_SERVICES` environment variable.

```bash
cf env foo-service

{
 "VCAP_SERVICES": {
  "p-config-server": [
   {
    "binding_name": null,
    "credentials": {
     "access_token_uri": "https://p-spring-cloud-services.uaa.run.pivotal.io/oauth/token",
     "client_id": "p-config-server-9c1a682a-089d-49be-97e0-1ad1f83366a1",
     "client_secret": "a2MRVZLjUyCy",
     "uri": "https://config-7e4b48ae-efd6-4773-b377-c8d1edf5ced6.cfapps.io"
    },
    "instance_name": "config-server",
    "label": "p-config-server",
    "name": "config-server",
    "plan": "trial",
    "provider": null,
    "syslog_drain_url": null,
    "tags": [
     "configuration",
     "spring-cloud"
    ],
    "volume_mounts": []
   }
  ]
 }
}
```

We can see that the communication between the `config-server` service instance and its client applications is via OAuth 2.0 protocol. The client must obtain first an OAuth 2.0 token in order to make requests directly against the `config-server`.
The included `spring-cloud-services-starter-config-client` dependency handles these OAuth 2.0 requests behind the scene for us.

```bash
curl https://p-config-server-9c1a682a-089d-49be-97e0-1ad1f83366a1:a2MRVZLjUyCy@p-spring-cloud-services.uaa.run.pivotal.io/oauth/token -d grant_type=client_credentials
```

We get an access token like this. Note that according to the response the token is only valid for 29 seconds.  

```json
{
    "access_token":"eyJhbGciOiJSUzI1NiIsImtpZCI6ImxlZ2FjeS10b2tlbi1rZXkiLCJ0eXAiOiJKV1QifQ.eyJqdGkiOiIzN2MxZjczNTMwMDc0OTQ2ODFjZTIxNzFiZWIxZDllZiIsInN1YiI6InAtY29uZmlnLXNlcnZlci05YzFhNjgyYS0wODlkLTQ5YmUtOTdlMC0xYWQxZjgzMzY2YTEiLCJhdXRob3JpdGllcyI6WyJwLWNvbmZpZy1zZXJ2ZXIuN2U0YjQ4YWUtZWZkNi00NzczLWIzNzctYzhkMWVkZjVjZWQ2LnJlYWQiXSwic2NvcGUiOlsicC1jb25maWctc2VydmVyLjdlNGI0OGFlLWVmZDYtNDc3My1iMzc3LWM4ZDFlZGY1Y2VkNi5yZWFkIl0sImNsaWVudF9pZCI6InAtY29uZmlnLXNlcnZlci05YzFhNjgyYS0wODlkLTQ5YmUtOTdlMC0xYWQxZjgzMzY2YTEiLCJjaWQiOiJwLWNvbmZpZy1zZXJ2ZXItOWMxYTY4MmEtMDg5ZC00OWJlLTk3ZTAtMWFkMWY4MzM2NmExIiwiYXpwIjoicC1jb25maWctc2VydmVyLTljMWE2ODJhLTA4OWQtNDliZS05N2UwLTFhZDFmODMzNjZhMSIsImdyYW50X3R5cGUiOiJjbGllbnRfY3JlZGVudGlhbHMiLCJyZXZfc2lnIjoiOWZlZjNmZDYiLCJpYXQiOjE1MzIxODkwMTQsImV4cCI6MTUzMjE4OTA0NCwiaXNzIjoiaHR0cHM6Ly9wLXNwcmluZy1jbG91ZC1zZXJ2aWNlcy51YWEucnVuLnBpdm90YWwuaW8vb2F1dGgvdG9rZW4iLCJ6aWQiOiIwNDlmYjNmOC1mOTc2LTQ1YTQtOWVhMy0wMDRkNDU2ZDRlMzgiLCJhdWQiOlsicC1jb25maWctc2VydmVyLTljMWE2ODJhLTA4OWQtNDliZS05N2UwLTFhZDFmODMzNjZhMSIsInAtY29uZmlnLXNlcnZlci43ZTRiNDhhZS1lZmQ2LTQ3NzMtYjM3Ny1jOGQxZWRmNWNlZDYiXX0.NDKKceOUv_WigMox38xoE7VuDRi6uz3zddFuu7F8Wz7UtLWgZ3wgLyzeKfT2f4gljgEOqT5XU3CJRazVHpbCdgTkhb4M37_sgFnHOZPYFgjN3GneSfT3ZLNhKyf36OTxbPA6sIgBYfFc1i6NDyG5y8CeLdDxEBkm5GMDtlvP-rptDhokeSlnJ6J1M6pJQZd7wYDqxZPfpvLesPE--yjUmRDO4Np8dF5DSOhSGM1El5HmCFJhy6lze1fizaLsGtbnqPIbZLlceENoullvHT-dt0w_Wxf9ErcUp_DIi3BMo5fUSCVyNLfTG965aye16_9v2IL9A08CnThxcCWZvk6Z1A",
    "token_type":"bearer",
    "expires_in":29,
    "scope":"p-config-server.7e4b48ae-efd6-4773-b377-c8d1edf5ced6.read",
    "jti":"37c1f7353007494681ce2171beb1d9ef"
}
```

And with this token the client application can send direct requests to the `config-server`: 

```bash
http https://config-7e4b48ae-efd6-4773-b377-c8d1edf5ced6.cfapps.io/foo-service/default  'Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6ImxlZ2FjeS10b2tlbi1rZXkiLCJ0eXAiOiJKV1QifQ.eyJqdGkiOiIyMTE3NGUwMjNiNzU0ZTg3YWYxNTczYmUzNTI3YjJmYSIsInN1YiI6InAtY29uZmlnLXNlcnZlci05YzFhNjgyYS0wODlkLTQ5YmUtOTdlMC0xYWQxZjgzMzY2YTEiLCJhdXRob3JpdGllcyI6WyJwLWNvbmZpZy1zZXJ2ZXIuN2U0YjQ4YWUtZWZkNi00NzczLWIzNzctYzhkMWVkZjVjZWQ2LnJlYWQiXSwic2NvcGUiOlsicC1jb25maWctc2VydmVyLjdlNGI0OGFlLWVmZDYtNDc3My1iMzc3LWM4ZDFlZGY1Y2VkNi5yZWFkIl0sImNsaWVudF9pZCI6InAtY29uZmlnLXNlcnZlci05YzFhNjgyYS0wODlkLTQ5YmUtOTdlMC0xYWQxZjgzMzY2YTEiLCJjaWQiOiJwLWNvbmZpZy1zZXJ2ZXItOWMxYTY4MmEtMDg5ZC00OWJlLTk3ZTAtMWFkMWY4MzM2NmExIiwiYXpwIjoicC1jb25maWctc2VydmVyLTljMWE2ODJhLTA4OWQtNDliZS05N2UwLTFhZDFmODMzNjZhMSIsImdyYW50X3R5cGUiOiJjbGllbnRfY3JlZGVudGlhbHMiLCJyZXZfc2lnIjoiOWZlZjNmZDYiLCJpYXQiOjE1MzIxODk4MTgsImV4cCI6MTUzMjE4OTg0OCwiaXNzIjoiaHR0cHM6Ly9wLXNwcmluZy1jbG91ZC1zZXJ2aWNlcy51YWEucnVuLnBpdm90YWwuaW8vb2F1dGgvdG9rZW4iLCJ6aWQiOiIwNDlmYjNmOC1mOTc2LTQ1YTQtOWVhMy0wMDRkNDU2ZDRlMzgiLCJhdWQiOlsicC1jb25maWctc2VydmVyLTljMWE2ODJhLTA4OWQtNDliZS05N2UwLTFhZDFmODMzNjZhMSIsInAtY29uZmlnLXNlcnZlci43ZTRiNDhhZS1lZmQ2LTQ3NzMtYjM3Ny1jOGQxZWRmNWNlZDYiXX0.RE0MBHDPolEeiz9Y9R0kb1_rzdyCr1c7AQA3zz4qiPJDW0DsnOFa-_GS-z7eyrz4QCanvFmd9m9WpRsh_Y4bnATPiYclqdXy65-Ziu3FKZka8Tv8WQ7dNOM520d2Dn6PcDapz8hJ7zoPJ9RxS2daZVgNkhzgFQOLxzk6t5coecV8ccN4-lbF3yK_E0aHe2QsIyZSeiQoBWsKC1r_8wZSgVbCFcQ7WkcSS9MIQ9SOxyLpIKsxjDj2JASw7MNID5iBJ8J_-0FfNtu710j7ZhHOh_4Xy9r399ie34NvdQKHTjqoctBjEKUD6CqaUwuYQWIXNj__ox44ex3RXAEq0d92Jg'
```

```json
{
    "label": null,
    "name": "foo-service",
    "profiles": [
        "default"
    ],
    "propertySources": [
        {
            "name": "https://github.com/altfatterz/config-repo.git/foo-service.yml",
            "source": {
                "foo.message": "Hallo",
                "foo.secret": "Gruetzi"
            }
        }
    ],
    "state": null,
    "version": "3635a8a138bcac086c66e55013ae1595aded0045"
}
```



