---
layout: post
title: Build images with Cloud Native Buildpacks
tags: [buildpacks, pack, docker, springboot]
---

Cloud Native Buildpacks (CNB) is a [specification](https://github.com/buildpacks/spec/blob/main/buildpack.md) to transform your application source code into an [OCI](https://github.com/opencontainers/image-spec/blob/master/spec.md) image. 
It was initiated by Pivotal and Heroku in 2018, and recently moved from Sandbox to Incubation by the [CNCF](https://www.cncf.io/blog/2020/11/18/toc-approves-cloud-native-buildpacks-from-sandbox-to-incubation/).
If you are interested in how it compares to other popular alternatives like [Jib](https://github.com/GoogleContainerTools/jib) or [s2i](https://github.com/openshift/source-to-image) head over to [https://buildpacks.io/features/](https://buildpacks.io/features/)

A **buildpack** inspects your application code and transforms the source code into a runnable app image. 
It has an *auto-detection* feature, for example a **Maven buildpack** looks for pom.xml, an **NPM buildpack** looks for a package.json, etc.
There is always a list of ordered buildpacks which are applied on your application source code, which are encapsulated in an image called the **builder**.
The **builder** contains the ordered list of buildpacks together with the base image to use for the app container image.       

The fastest way to get started with CNB is with the [pack](https://github.com/buildpacks/pack) reference CLI. 
`Pack` uses Docker to run the build process in isolated environment. 

After [installing](https://buildpacks.io/docs/tools/pack/) the pack CLI let's use it on one example:

```bash
$ git clone http://github.com/altfatterz/buildpacks-demo
$ cd buildpacks-demo
$ pack build buildpacks-demo --builder paketobuildpacks/builder:base
```

By default, the image is uploaded to the local Docker daemon, but with `--publish` parameter we can specify a registry.

```bash
$ docker images

REPOSITORY                 TAG        IMAGE ID       CREATED        SIZE
paketobuildpacks/run       base-cnb   4347963b10a1   5 days ago     106MB
paketobuildpacks/builder   base       03aa716c9552   41 years ago   558MB
buildpacks-demo            latest     a29f2833a7a1   41 years ago   278MB 
```

We can see that the `paketobuildpacks/builder` and the run-image `paketobuildpacks/run` were pulled from the [dockerhub](https://hub.docker.com/search?q=paketobuildpacks).
If you are confused by the `41 years ago` timestamp it has to do with to allow reproducible builds. Here you can find more information about the details: [Time Travel with Pack](https://medium.com/buildpacks/time-travel-with-pack-e0efd8bf05db) 

[Packeto Buildpacks](https://paketo.io/) is an implementation of [Cloud Native Buildpacks](https://buildpacks.io/).

With `pack` we can inspect the built app image. Here we can see that with the used builder which buildpacks were applied to our source code and which was the base image. 

```bash
$ pack inspect-image buildpacks-demo
...
Base Image:
  Reference: 4347963b10a16e72afa854b12834d41835f4347d7a92f5428bb47c0238464c95
  Top Layer: sha256:8cc74c487427203dd7ff8b5a615b63f9178117a42a4a966f3606fd51746e0fc5

Run Images:
  index.docker.io/paketobuildpacks/run:base-cnb
  gcr.io/paketo-buildpacks/run:base-cnb

Buildpacks:
  ID                                         VERSION
  paketo-buildpacks/ca-certificates          1.0.1
  paketo-buildpacks/bellsoft-liberica        6.0.0
  paketo-buildpacks/maven                    3.2.1
  paketo-buildpacks/executable-jar           3.1.3
  paketo-buildpacks/apache-tomcat            3.1.0
  paketo-buildpacks/dist-zip                 2.2.2
  paketo-buildpacks/spring-boot              3.5.0
... 
``` 

[Dive](https://github.com/wagoodman/dive) is also a great tool to get a view into the file system of each layer.

<p><img src="/images/2020-12-26/dive.png" alt="Dive" /></p>

We can inspect also the builder using

```bash
$ pack inspect-builder paketobuildpacks/builder:base
```

where we can see that it supports multiple programming languages.

Next to `detect` and `build` there are 3 more stages in the lifecycle:

1. `detector` - Detects the app type and produces a build plan.
2. `analyzer` - Restores layer metadata from the previous image and from the cache.
3. `restorer` - Restores cached layers.
4. `builder` - Executes buildpacks.
5. `exporter` - Creates an image and caches layers.

This ensures that after the first build the builds are much faster, but also safer since only the layers that need to change are replaced.

Spring Boot also has support for the [CNB](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-container-images-buildpacks). The [Maven](https://docs.spring.io/spring-boot/docs/2.4.1/maven-plugin/reference/htmlsingle/#build-image) and [Gradle](https://docs.spring.io/spring-boot/docs/2.4.1/gradle-plugin/reference/htmlsingle/#build-image) plugins are using the Packeto Buildpacks implementation.

```bash
$ mvn spring-boot:build-image
```

Below you can explore what is happening in each lifecycle: `DETECTING`, `ANALYZING`, `RESTORING`, `BUILDING` and `EXPORTING`.
The image is based on the `docker.io/paketobuildpacks/run:base-cnb` run image, which is a minimal Paketo stack based on Ubuntu 18.04.
From the logs we can see that from the 18 available buildpacks 5 are applied to the app source code. The
`paketo-buildpacks/ca-certificates` adds CA certificates to the system truststore at build and runtime, the `paketo-buildpacks/bellsoft-liberica` requests that a JRE be installed.
In the end `paketo-buildpacks/spring-boot` contributes Spring Boot dependency information and slices an application into multiple layers. 

```bash
[INFO] Building image 'docker.io/library/buildpacks-demo:0.0.1-SNAPSHOT'
[INFO]
[INFO]  > Pulling builder image 'docker.io/paketobuildpacks/builder:base' 100%
[INFO]  > Pulled builder image 'paketobuildpacks/builder@sha256:984a3684db80a6d53214b81a9f21c31529bede5b447d6d6d82d94cd6734d2424'
[INFO]  > Pulling run image 'docker.io/paketobuildpacks/run:base-cnb' 100%
[INFO]  > Pulled run image 'paketobuildpacks/run@sha256:f393fa2927a2619a10fc09bb109f822d20df909c10fed4ce3c36fad313ea18e3'
[INFO]  > Executing lifecycle version v0.10.1
[INFO]  > Using build cache volume 'pack-cache-89b2691a0bb8.build'
[INFO]
[INFO]  > Running creator
[INFO]     [creator]     ===> DETECTING
[INFO]     [creator]     5 of 18 buildpacks participating
[INFO]     [creator]     paketo-buildpacks/ca-certificates   1.0.1
[INFO]     [creator]     paketo-buildpacks/bellsoft-liberica 6.0.0
[INFO]     [creator]     paketo-buildpacks/executable-jar    3.1.3
[INFO]     [creator]     paketo-buildpacks/dist-zip          2.2.2
[INFO]     [creator]     paketo-buildpacks/spring-boot       3.5.0
[INFO]     [creator]     ===> ANALYZING
[INFO]     [creator]     Previous image with name "docker.io/library/buildpacks-demo:0.0.1-SNAPSHOT" not found
[INFO]     [creator]     ===> RESTORING
[INFO]     [creator]     ===> BUILDING
[INFO]     [creator]
[INFO]     [creator]     Paketo CA Certificates Buildpack 1.0.1
[INFO]     [creator]       https://github.com/paketo-buildpacks/ca-certificates
[INFO]     [creator]       Launch Helper: Contributing to layer
[INFO]     [creator]         Creating /layers/paketo-buildpacks_ca-certificates/helper/exec.d/ca-certificates-helper
[INFO]     [creator]         Writing profile.d/helper
[INFO]     [creator]
[INFO]     [creator]     Paketo BellSoft Liberica Buildpack 6.0.0
[INFO]     [creator]       https://github.com/paketo-buildpacks/bellsoft-liberica
[INFO]     [creator]       Build Configuration:
[INFO]     [creator]         $BP_JVM_VERSION              11.*            the Java version
[INFO]     [creator]       Launch Configuration:
[INFO]     [creator]         $BPL_JVM_HEAD_ROOM           0               the headroom in memory calculation
[INFO]     [creator]         $BPL_JVM_LOADED_CLASS_COUNT  35% of classes  the number of loaded classes in memory calculation
[INFO]     [creator]         $BPL_JVM_THREAD_COUNT        250             the number of threads in memory calculation
[INFO]     [creator]         $JAVA_TOOL_OPTIONS                           the JVM launch flags
[INFO]     [creator]       BellSoft Liberica JRE 11.0.9: Contributing to layer
[INFO]     [creator]         Downloading from https://github.com/bell-sw/Liberica/releases/download/11.0.9.1+1/bellsoft-jre11.0.9.1+1-linux-amd64.tar.gz
[INFO]     [creator]         Verifying checksum
[INFO]     [creator]         Expanding to /layers/paketo-buildpacks_bellsoft-liberica/jre
[INFO]     [creator]         Adding 138 container CA certificates to JVM truststore
[INFO]     [creator]         Writing env.launch/BPI_APPLICATION_PATH.default
[INFO]     [creator]         Writing env.launch/BPI_JVM_CACERTS.default
[INFO]     [creator]         Writing env.launch/BPI_JVM_CLASS_COUNT.default
[INFO]     [creator]         Writing env.launch/BPI_JVM_SECURITY_PROVIDERS.default
[INFO]     [creator]         Writing env.launch/JAVA_HOME.default
[INFO]     [creator]         Writing env.launch/MALLOC_ARENA_MAX.default
[INFO]     [creator]       Launch Helper: Contributing to layer
[INFO]     [creator]         Creating /layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/active-processor-count
[INFO]     [creator]         Creating /layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/java-opts
[INFO]     [creator]         Creating /layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/link-local-dns
[INFO]     [creator]         Creating /layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/memory-calculator
[INFO]     [creator]         Creating /layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/openssl-certificate-loader
[INFO]     [creator]         Creating /layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/security-providers-configurer
[INFO]     [creator]         Creating /layers/paketo-buildpacks_bellsoft-liberica/helper/exec.d/security-providers-classpath-9
[INFO]     [creator]         Writing profile.d/helper
[INFO]     [creator]       JVMKill Agent 1.16.0: Contributing to layer
[INFO]     [creator]         Downloading from https://github.com/cloudfoundry/jvmkill/releases/download/v1.16.0.RELEASE/jvmkill-1.16.0-RELEASE.so
[INFO]     [creator]         Verifying checksum
[INFO]     [creator]         Copying to /layers/paketo-buildpacks_bellsoft-liberica/jvmkill
[INFO]     [creator]         Writing env.launch/JAVA_TOOL_OPTIONS.append
[INFO]     [creator]         Writing env.launch/JAVA_TOOL_OPTIONS.delim
[INFO]     [creator]       Java Security Properties: Contributing to layer
[INFO]     [creator]         Writing env.launch/JAVA_SECURITY_PROPERTIES.default
[INFO]     [creator]         Writing env.launch/JAVA_TOOL_OPTIONS.append
[INFO]     [creator]         Writing env.launch/JAVA_TOOL_OPTIONS.delim
[INFO]     [creator]
[INFO]     [creator]     Paketo Executable JAR Buildpack 3.1.3
[INFO]     [creator]       https://github.com/paketo-buildpacks/executable-jar
[INFO]     [creator]         Writing env.launch/CLASSPATH.delim
[INFO]     [creator]         Writing env.launch/CLASSPATH.prepend
[INFO]     [creator]       Process types:
[INFO]     [creator]         executable-jar: java org.springframework.boot.loader.JarLauncher
[INFO]     [creator]         task:           java org.springframework.boot.loader.JarLauncher
[INFO]     [creator]         web:            java org.springframework.boot.loader.JarLauncher
[INFO]     [creator]
[INFO]     [creator]     Paketo Spring Boot Buildpack 3.5.0
[INFO]     [creator]       https://github.com/paketo-buildpacks/spring-boot
[INFO]     [creator]       Creating slices from layers index
[INFO]     [creator]         dependencies
[INFO]     [creator]         spring-boot-loader
[INFO]     [creator]         snapshot-dependencies
[INFO]     [creator]         application
[INFO]     [creator]       Launch Helper: Contributing to layer
[INFO]     [creator]         Creating /layers/paketo-buildpacks_spring-boot/helper/exec.d/spring-cloud-bindings
[INFO]     [creator]         Writing profile.d/helper
[INFO]     [creator]       Web Application Type: Contributing to layer
[INFO]     [creator]         Servlet web application detected
[INFO]     [creator]         Writing env.launch/BPL_JVM_THREAD_COUNT.default
[INFO]     [creator]       Spring Cloud Bindings 1.7.0: Contributing to layer
[INFO]     [creator]         Downloading from https://repo.spring.io/release/org/springframework/cloud/spring-cloud-bindings/1.7.0/spring-cloud-bindings-1.7.0.jar
[INFO]     [creator]         Verifying checksum
[INFO]     [creator]         Copying to /layers/paketo-buildpacks_spring-boot/spring-cloud-bindings
[INFO]     [creator]       4 application slices
[INFO]     [creator]       Image labels:
[INFO]     [creator]         org.opencontainers.image.title
[INFO]     [creator]         org.opencontainers.image.version
[INFO]     [creator]         org.springframework.boot.spring-configuration-metadata.json
[INFO]     [creator]         org.springframework.boot.version
[INFO]     [creator]     ===> EXPORTING
[INFO]     [creator]     Adding layer 'paketo-buildpacks/ca-certificates:helper'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/bellsoft-liberica:helper'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/bellsoft-liberica:java-security-properties'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/bellsoft-liberica:jre'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/bellsoft-liberica:jvmkill'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/executable-jar:class-path'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/spring-boot:helper'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/spring-boot:spring-cloud-bindings'
[INFO]     [creator]     Adding layer 'paketo-buildpacks/spring-boot:web-application-type'
[INFO]     [creator]     Adding 5/5 app layer(s)
[INFO]     [creator]     Adding layer 'launcher'
[INFO]     [creator]     Adding layer 'config'
[INFO]     [creator]     Adding layer 'process-types'
[INFO]     [creator]     Adding label 'io.buildpacks.lifecycle.metadata'
[INFO]     [creator]     Adding label 'io.buildpacks.build.metadata'
[INFO]     [creator]     Adding label 'io.buildpacks.project.metadata'
[INFO]     [creator]     Adding label 'org.opencontainers.image.title'
[INFO]     [creator]     Adding label 'org.opencontainers.image.version'
[INFO]     [creator]     Adding label 'org.springframework.boot.spring-configuration-metadata.json'
[INFO]     [creator]     Adding label 'org.springframework.boot.version'
[INFO]     [creator]     Setting default process type 'web'
[INFO]     [creator]     *** Images (54bf7c7f30b1):
[INFO]     [creator]           docker.io/library/buildpacks-demo:0.0.1-SNAPSHOT
[INFO]
[INFO] Successfully built image 'docker.io/library/buildpacks-demo:0.0.1-SNAPSHOT'
[INFO]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  29.156 s
[INFO] Finished at: 2020-12-25T19:10:19+01:00
[INFO] ------------------------------------------------------------------------
```

Now we can run our application using:

```bash
$ docker run --rm -p 8080:8080 buildpacks-demo:0.0.1-SNAPSHOT
$ http :8080
Hello, Buildpacker!
```

With `pack` is possible to `rebase` the image to a version pinned run image. This is useful for example when there is a security issue with the run image and with this command we can change only that OS layer in the image.

```bash
$ pack rebase buildpacks-demo:latest --run-image paketobuildpacks/run:1.0.11-base-cnb

1.0.11-base-cnb: Pulling from paketobuildpacks/run
Digest: sha256:f393fa2927a2619a10fc09bb109f822d20df909c10fed4ce3c36fad313ea18e3
Status: Image is up to date for paketobuildpacks/run:1.0.11-base-cnb
Rebasing buildpacks-demo:latest on run image paketobuildpacks/run:1.0.11-base-cnb
*** Images (7c8abb9b40d2):
      buildpacks-demo:latest
Rebased Image: 7c8abb9b40d2f50b556efa003642ca5d00a24ec76fb91efc82c52d37887401a5
Successfully rebased image buildpacks-demo:latest
```

There is another tool `kpack` which running as a service on Kubernetes could do this `rebase` on all of your affected images, but that is a topic for another blog post :)