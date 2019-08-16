---
layout: post
title: Dockerizing Spring Boot apps with Jib
tags: [docker, jib, springboot]
---

[Jib](https://github.com/GoogleContainerTools/jib) is my preferred tool when it comes to dockerizing Spring Boot applications.
Jib is pure Java which can create a docker image without Dockerfile and without a running Docker daemon by automatically uploading the image to a remote Docker Registry.
It can also create a Docker image using a local Docker daemon.

### Setup

Jib comes as a [Maven](https://github.com/GoogleContainerTools/jib/tree/master/jib-maven-plugin) and [Gradle](https://github.com/GoogleContainerTools/jib/tree/master/jib-gradle-plugin) plugin. In this blog post we will use Maven. 

```xml
<build>
    ...
    <plugin>
        <groupId>com.google.cloud.tools</groupId>
        <artifactId>jib-maven-plugin</artifactId>
        <version>1.4.0</version>
    </plugin>
    ...
</build>
```

### Without Docker daemon

With the following command a docker image is created and then uploaded to Google Container Registry.

```bash
$ mvn clean install jib:build -Dimage=eu.gcr.io/altfatterz/docker-jib-demo
```

To examine network connectivity issues we can use:

```bash
$ mvn -X -DjibSerialize=true
```

In this case Jib is using the [docker_credential_helper](https://cloud.google.com/container-registry/docs/advanced-authentication#docker_credential_helper)
to retrieve the credentials for Google Container Registry. Using a `Docker credential helper` is the preferred way to set the credentials to a Container Registry instead of defining these credentials in your Maven settings.
This approach is basically using `gcloud`, which is the recommended way to connect to a Google Container Registry, you only need to run `gcloud auth configure-docker` to set this up.

There is another more advanced way, using the `docker-credential-gcr` which you can setup following the `Standalone Docker credential helper` section of [this](https://cloud.google.com/container-registry/docs/advanced-authentication#docker_credential_helper) documentation.

For AWS, Jib would use [amazon-ecr-credential-helper](https://github.com/awslabs/amazon-ecr-credential-helper) and for Azure Container Registry [acr-docker-credential-helper](https://github.com/Azure/acr-docker-credential-helper) Docker credential helper.   

### With Docker daemon

Jib can also use the local Docker daemon:

```bash
$ mvn clean install jib:dockerBuild -Dimage=altfatterz/docker-jib-demo
```

By default Jib is using ["Distroless" Docker Images](https://github.com/GoogleContainerTools/distroless) as the base image. These are language focused docker images without any package managers, shells or any other programs you would expect to find in a standard Linux distribution.  

You can easily change the base Docker image: 

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>${jib.version}</version>
    <configuration>
        <from>
            <image>openjdk:8-jre-alpine</image>
        </from>                
    </configuration>
</plugin>
```

### Tags

You can easily tag the docker images, for example using the maven project version:

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>${jib.version}</version>
    <configuration>
        <to>
            <tags>${project.version}</tags>
        </to>           
    </configuration>
</plugin>
```

Tags make it easy to manage versions of the docker images, an example using Google Container Registry 

<p><img src="/images/2019-08-16/gcr-docker-jib-demo.png" alt="gcr-docker-jib-demo" /></p>

or with command line:

```bash
$ gcloud container images list-tags eu.gcr.io/altfatterz/docker-jib-demo

DIGEST        TAGS                   TIMESTAMP
26332d31fd4a  0.0.3-SNAPSHOT,latest  2019-08-16T16:48:03
9cf7c00784eb  0.0.2                  2019-08-16T16:46:11
9f8880641da7  0.0.1                  2019-08-16T16:45:23
```

### Multi-layered images

Jib separates your application into multiple layers, as you can see with the following command:

```
$ docker inspect altfatterz/docker-jib-demo

[
    {
        "Id": "sha256:c5bcbd3183fb47c0ad709a787dc72af2dbdd958ce8f7fb9af8a94a0ea0bcf2e7",
        "RepoTags": [
            "docker-jib-demo:0.0.1-SNAPSHOT"
        ],
        ...
        "Config": {
            ...
            "Env": [
                "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
                "SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt",
                "JAVA_VERSION=8u222"
            ],
            ...
            "Entrypoint": [
                "java",
                "-cp",
                "/app/resources:/app/classes:/app/libs/*",
                "com.example.DockerJibDemoApplication"
            ],
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:932da51564135c98a49a34a193d6cd363d8fa4184d957fde16c9d8527b3f3b02",
                "sha256:dffd9992ca398466a663c87c92cfea2a2db0ae0cf33fcb99da60eec52addbfc5",
                "sha256:6189abe095d53c1c9f2bfc8f50128ee876b9a5d10f9eda1564e5f5357d6ffe61",
                "sha256:24917f2870609331183d9702704a5555e4cdd1f1d36aae117637bcb415f76ff3",
                "sha256:66f04fc98f17b361d27144cdc9dc20597e43c969cad47503a95b5f29ae2074a0",
                "sha256:289eb806ed2e1a0c9a66a2fa539543c4b583ac483209c12b393664af4c18d396",
                "sha256:41ecf942828c5c4adbb1d11d52ae7941f0ad7cfa3699fc0afebdd3fd71669062"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]
```

or with the following command:

```bash
$ docker history docker-jib-demo:0.0.1-SNAPSHOT

IMAGE               CREATED             CREATED BY               SIZE                COMMENT
c5bcbd3183fb        5 hours ago         jib-maven-plugin:1.4.0   1.03kB              classes
<missing>           5 hours ago         jib-maven-plugin:1.4.0   1B                  resources
<missing>           5 hours ago         jib-maven-plugin:1.4.0   16.7MB              dependencies
<missing>           49 years ago        bazel build ...          106MB
<missing>           49 years ago        bazel build ...          1.93MB
<missing>           49 years ago        bazel build ...          15.1MB
<missing>           49 years ago        bazel build ...          1.82MB
```

This makes sure that when only classes are changed only the classes layer will be rebuilt, causing very fast build times. 

### Jib Image Built 49 Years Ago?

You need to specify the following, otherwise the `CREATED` field will be `49 years ago`

```xml
<plugin>
    <groupId>com.google.cloud.tools</groupId>
    <artifactId>jib-maven-plugin</artifactId>
    <version>${jib.version}</version>
    <configuration>
        <container>
            <useCurrentTimestamp>true</useCurrentTimestamp>
        </container>          
    </configuration>
</plugin>
```

You can find the used example here: [http://github.com/altfatterz/docker-jib-demo.git](http://github.com/altfatterz/docker-jib-demo.git)