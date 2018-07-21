---
layout: post
title: Spring Cloud Services - Config Server
tags: [springcloudservices, springcloud, spring]
---


Inside the `~/.spring-cloud` folder `configserver.yml`

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

Inside the `config-repo` create a `foo-service.yml` file with the content

```yaml
foo:
  message: Hallo
  secret: '{cipher}6050a8af7826c89dcd30c9046f484a5a169d27beb40a67db72440474c5cc6d77'
```

The `foo.secret` was encrypted with the s3cr3t encryption key using the `spring` CLI client (with Spring Cloud CLI extensions installed)

```bash
spring encrypt Gruetzi --key s3cr3t
```

Set the `ENCRYPT_KEY` environment variable when starting the `config server` 

```bash
export ENCRYPT_KEY=s3cr3t
``` 

And start up the `config server` using

```bash
spring cloud configserver
```

Verify that the config server knows about the external configuration of the `foo-service` 

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

After starting the  `foo-service` application you should see in the logs:

```
[           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at : http://localhost:8888
```

The `foo-service` is using `config-first bootstrap` and with the `spring.cloud.config.uri` locates the config server which is set to `http://localhost:8888` by default.

With a simple controller we print out the `foo.message` and `foo.secret` properties which we can verify with

```
http :8080 -a user:password
Hallo/Gruetzi
```

So far so good. Let's setup this little example on a `Pivotal Cloud Foundry` where we will use the [Config Server](http://docs.pivotal.io/spring-cloud-services/1-5/common/config-server/index.html) from [Spring Cloud Service](http://docs.pivotal.io/spring-cloud-services/1-5/common/index.html).

```xml
<dependency>
    <groupId>io.pivotal.spring.cloud</groupId>
    <artifactId>spring-cloud-services-starter-config-client</artifactId>
</dependency>
```

Make sure you install first the [CloudFoundry CLI](https://github.com/cloudfoundry/cli) and create a [Pivotal Web Services](https://run.pivotal.io/) account.

After we pushed the [config-rep](https://github.com/altfatterz/config-repo) to GitHub we can configure a `config-server` instance with this shared git repository. 
We will use the `p-config-server` service with the `trial` plan as described which we retrieved from the marketplace via `cf marketplace`
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

Check that the service is created:

```bash
cf service config-server

Showing info of service config-server in org altfatterz-org / space development as altfatterz@gmail.com...

name:            config-server
service:         p-config-server
tags:
plan:            trial
description:     Config Server for Spring Cloud Applications
documentation:
dashboard:       https://spring-cloud-service-broker.cfapps.io/dashboard/p-config-server/7e4b48ae-efd6-4773-b377-c8d1edf5ced6

This service is not currently shared.

There are no bound apps for this service.

Showing status of last operation from service config-server...

status:    create succeeded
message:
started:   2018-07-20T14:33:39Z
updated:   2018-07-20T14:35:46Z
```

Next we need to set the encrypt key for the `config-server`. By default the `config-server` will decrypt the secrets and return the decrypted secrets to the client applications like in our case the to the `foo-service`. It is important that you protect the config server and have https   
If you want that the clients to decrypt the secrets locally you need to set the `spring.cloud.config.server.encrypt.enabled=false` in the `bootstrap.yml`         

```bash
cf update-service config-server -c '{"count":1,"git":{"uri":"https://github.com/altfatterz/config-repo.git"}, "encrypt":{"key":"s3cr3t"} }'
```

If you know how to set this easier, please let me know if the comments.

Then in the `foo-service` add the following `mainfest.yml` file

```yaml
applications:
- name: foo-service
  host: foo-service-${random-word}
  memory: 756M
  instances: 1
  path: target/foo-service-0.0.1-SNAPSHOT.jar
  services:
  - config-server
```

and deploy the service using `cf push` inside the `foo-service` directory.


```bash
http :8888/foo-service/default
```

Verify the app status with `cf service foo-service`

```bash
name:              foo-service
requested state:   started
instances:         1/1
usage:             756M x 1 instances
routes:            foo-service.cfapps.io
last uploaded:     Fri 20 Jul 16:57:49 CEST 2018
stack:             cflinuxfs2
buildpack:         client-certificate-mapper=1.6.0_RELEASE container-security-provider=1.14.0_RELEASE java-buildpack=v4.13-offline-https://github.com/cloudfoundry/java-buildpack.git#4644847 java-main java-opts
                   java-security jvmkill-agent=1.16.0_RELEASE open-jdk-...

     state     since                  cpu    memory           disk           details
#0   running   2018-07-20T15:13:35Z   0.3%   250.6M of 756M   152.1M of 1G
```

```bash
http foo-service.cfapps.io -a user:password

Hallo/Gruetzi
```

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

```bash
curl https://p-config-server-9c1a682a-089d-49be-97e0-1ad1f83366a1:a2MRVZLjUyCy@p-spring-cloud-services.uaa.run.pivotal.io/oauth/token -d grant_type=client_credentials
```

```json
{
    "access_token":"eyJhbGciOiJSUzI1NiIsImtpZCI6ImxlZ2FjeS10b2tlbi1rZXkiLCJ0eXAiOiJKV1QifQ.eyJqdGkiOiIzN2MxZjczNTMwMDc0OTQ2ODFjZTIxNzFiZWIxZDllZiIsInN1YiI6InAtY29uZmlnLXNlcnZlci05YzFhNjgyYS0wODlkLTQ5YmUtOTdlMC0xYWQxZjgzMzY2YTEiLCJhdXRob3JpdGllcyI6WyJwLWNvbmZpZy1zZXJ2ZXIuN2U0YjQ4YWUtZWZkNi00NzczLWIzNzctYzhkMWVkZjVjZWQ2LnJlYWQiXSwic2NvcGUiOlsicC1jb25maWctc2VydmVyLjdlNGI0OGFlLWVmZDYtNDc3My1iMzc3LWM4ZDFlZGY1Y2VkNi5yZWFkIl0sImNsaWVudF9pZCI6InAtY29uZmlnLXNlcnZlci05YzFhNjgyYS0wODlkLTQ5YmUtOTdlMC0xYWQxZjgzMzY2YTEiLCJjaWQiOiJwLWNvbmZpZy1zZXJ2ZXItOWMxYTY4MmEtMDg5ZC00OWJlLTk3ZTAtMWFkMWY4MzM2NmExIiwiYXpwIjoicC1jb25maWctc2VydmVyLTljMWE2ODJhLTA4OWQtNDliZS05N2UwLTFhZDFmODMzNjZhMSIsImdyYW50X3R5cGUiOiJjbGllbnRfY3JlZGVudGlhbHMiLCJyZXZfc2lnIjoiOWZlZjNmZDYiLCJpYXQiOjE1MzIxODkwMTQsImV4cCI6MTUzMjE4OTA0NCwiaXNzIjoiaHR0cHM6Ly9wLXNwcmluZy1jbG91ZC1zZXJ2aWNlcy51YWEucnVuLnBpdm90YWwuaW8vb2F1dGgvdG9rZW4iLCJ6aWQiOiIwNDlmYjNmOC1mOTc2LTQ1YTQtOWVhMy0wMDRkNDU2ZDRlMzgiLCJhdWQiOlsicC1jb25maWctc2VydmVyLTljMWE2ODJhLTA4OWQtNDliZS05N2UwLTFhZDFmODMzNjZhMSIsInAtY29uZmlnLXNlcnZlci43ZTRiNDhhZS1lZmQ2LTQ3NzMtYjM3Ny1jOGQxZWRmNWNlZDYiXX0.NDKKceOUv_WigMox38xoE7VuDRi6uz3zddFuu7F8Wz7UtLWgZ3wgLyzeKfT2f4gljgEOqT5XU3CJRazVHpbCdgTkhb4M37_sgFnHOZPYFgjN3GneSfT3ZLNhKyf36OTxbPA6sIgBYfFc1i6NDyG5y8CeLdDxEBkm5GMDtlvP-rptDhokeSlnJ6J1M6pJQZd7wYDqxZPfpvLesPE--yjUmRDO4Np8dF5DSOhSGM1El5HmCFJhy6lze1fizaLsGtbnqPIbZLlceENoullvHT-dt0w_Wxf9ErcUp_DIi3BMo5fUSCVyNLfTG965aye16_9v2IL9A08CnThxcCWZvk6Z1A",
    "token_type":"bearer",
    "expires_in":29,
    "scope":"p-config-server.7e4b48ae-efd6-4773-b377-c8d1edf5ced6.read",
    "jti":"37c1f7353007494681ce2171beb1d9ef"
}
```

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



