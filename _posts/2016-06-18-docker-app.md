---
layout: post
title: Docker App
tags: [docker]
---

In this blog post I would like to show you that the recently introduced Docker Native application (currently in private beta) makes it possible to configure a docker container with outside container storage without any workarounds on Mac or Windows, similarly how it works on Linux.

For storing data by applications running in a Docker container there are two options:

1. inside the container managed by docker. The problem with this is when the container is removed the data is also deleted.
2. outside the container in a directory on the host system which is mounted to be visible inside the container.

The Docker native application is currently in private beta which you can get after signing up at `https://beta.docker.com/`. You will receive an email with your private invitation code which is required when you start the application for the first time.
In this blog post I am going to focus on the Mac version since that is the platform I am using, but for Windows it is more or less similar.

<p><img src="/images/docker-for-mac.png" alt="Docker for Mac" /></p>

Docker for Mac application does not use VirtualBox, instead provisions a [HyperKit](https://github.com/docker/hyperkit) VM based on [Alpine Linux](http://www.alpinelinux.org/) which is running the Docker Engine. So with the Docker for Mac application you get only one VM and the app manages it as opposed to [`Docker Toolbox`](https://www.docker.com/products/docker-toolbox) where you could have created multiple VMs with `docker-machine`.

If you have used Docker Toolbox before make sure docker environment variables are unset (running the command `unset ${!DOCKER_*}`) since otherwise you will be talking to the Engine from the Toolbox, instead of the Engine from the Docker app.

For the example I will use MongoDB running in a Docker container. First you need to create a data directory, e.g. `/Users/Zoltan/mongodb/db-docker` and then you can start a `mongo` container like this using the official mongo [image](https://hub.docker.com/_/mongo/)

```sh
$ docker run -p 27007:27017 --name cool-mongo \
    -v /Users/Zoltan/mongodb/db-docker:/data/db mongo \
    mongod --smallfiles
```

The `-v /Users/Zoltan/mongodb/db-docker:/data/db` part of the command mounts the `/Users/Zoltan/mongodb/db-docker` directory form the host system as a `/data/db` inside the container.
You can name your container with the `--name` option. If you do not provide a name, Docker will generate a random one. The name of the container can also be used as a container id in other `docker` commands.
The `-p 27007:27017` part of the command bounds the 27017 port in the container to port 27007 on the host.

To verify that the docker container is running you can use:

```sh
$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                      NAMES
8a02110d327b        mongo               "/entrypoint.sh mongo"   20 seconds ago      Up 19 seconds       0.0.0.0:27017->27007/tcp   cool-mongo
```

Then you can easily connect to the running MongoDB via the `mongo` shell. You no longer need to figure out what is the ip address for your   docker machine.

```sh
$ mongo localhost:27007
```

Let's create a collection and insert a document:

```sh
$ db.foo.insert({})
```

After stopping, deleting and recreating the `cool-mongo` container you can verify that `foo` collection exists and contains the document previously created.

```sh
$ docker stop cool-mongo
$ docker rm -v cool-mongo
$ docker run -d -p 27007:27017 --name cool-mongo \
    -v /Users/Zoltan/mongodb/db-docker:/data/db mongo \
    mongod --smallfiles
$ mongo localhost:27007
MongoDB shell version: 3.2.1
connecting to: localhost:27007/test
> db.foo.find()
{ "_id" : ObjectId("5765aa52be07eeed3876d2c0") }
```

Go ahead an sign up for the Docker app, I think it provides a much better docker development experience on Mac. You can uninstall VirtualBox :)