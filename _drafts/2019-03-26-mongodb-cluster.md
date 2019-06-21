---
layout: post
title: Setup 3 node MongoDB replica set with Docker
tags: [mongodb]
---

We create a 3 node mongodb `ReplicaSet` using docker containers. 

##### Create a network

Since when creating a MongoDB [`replica set`](https://docs.mongodb.com/manual/replication/) we want to use container names to resolve the IP address we need to create a `user-defined` network, with the default `bridge` network it will not work. 

```bash
docker network create my-mongo-cluster
```

We are going to use the official mongo image from [DockerHub](https://hub.docker.com/). At the time of this writing the latest is [`mongo:4.0.8`](https://github.com/docker-library/mongo/blob/89f19dc16431025c00a4709e0da6d751cf94830f/4.0/Dockerfile) 

In the [`Dockerfile`](https://github.com/docker-library/mongo/blob/89f19dc16431025c00a4709e0da6d751cf94830f/4.0/Dockerfile) we see the declared volumes

```bash
VOLUME /data/db /data/configdb
```

We are using then [`bind mounts`](https://docs.docker.com/storage/bind-mounts/) to make sure that the data created by mongo container is not deleted when we remove the container. In the example below the `mongodb-cluster` directory is created on demand if it does not yet exist.
 
When creating the containers we reference the previously defined `user-defined` network with the `--net` option.  

```bash
docker container run -d --name mongo-node1 \
    -v $(pwd)/mongodb-cluster/data/db/node1:/data/db \
    -v $(pwd)/mongodb-cluster/data/configdb/node1:/data/configdb \
    -v $(pwd)/mongodb-cluster/config/node1:/etc/mongo \
    --net my-mongo-cluster \
    -p 27017:27017 \
    mongo:4.0.6 --config /etc/mongo/mongod.config
     
docker container run -d --name mongo-node2 \
    -v $(pwd)/mongodb-cluster/data/db/node2:/data/db \
    -v $(pwd)/mongodb-cluster/data/configdb/node2:/data/configdb \
    -v $(pwd)/mongodb-cluster/config/node2:/etc/mongo \
    --net my-mongo-cluster \
    -p 27018:27017 \
    mongo:4.0.6 --config /etc/mongo/mongod.config
 
docker container run -d --name mongo-node3 \
    -v $(pwd)/mongodb-cluster/data/db/node3:/data/db \
    -v $(pwd)/mongodb-cluster/data/configdb/node3:/data/configdb \
    -v $(pwd)/mongodb-cluster/config/node3:/etc/mongo \
    --net my-mongo-cluster \
    -p 27019:27017 \
    mongo:4.0.6 --config /etc/mongo/mongod.config
    
    ```

##### Verify that they are up and running

```bash
docker ps -a
```

```bash
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                 NAMES
dae526ee43c8        mongo:4.0.6         "docker-entrypoint.s…"   About an hour ago   Up 34 minutes       27017/tcp, 0.0.0.0:27019->27019/tcp   mongo-node3
af4623782ce5        mongo:4.0.6         "docker-entrypoint.s…"   About an hour ago   Up 24 minutes       27017/tcp, 0.0.0.0:27018->27018/tcp   mongo-node2
caac34cf1e57        mongo:4.0.6         "docker-entrypoint.s…"   About an hour ago   Up 7 minutes        0.0.0.0:27017->27017/tcp              mongo-node1
```

##### Initialize the `ReplicatSet`
 

```bash
docker exec -it mongo-node1 bash
mongo
```

```bash
config = {
      "_id" : "demo",
      "members" : [
          {
              "_id" : 0,
              "host" : "mongo-node1:27017"
          },
          {
              "_id" : 1,
              "host" : "mongo-node2:27018"
          },
          {
              "_id" : 2,
              "host" : "mongo-node3:27019"
          }
      ]
  }

rs.initiate(config)
```


##### Check the status

```bash
demo:PRIMARY> rs.status()
```

#### Client connection

```bash
brew install mongo
``` 

```bash
mongo --host demo/mongo-node1:27017,mongo-node2:27018,mongo-node3:27019 test
```

Trying to connect with:

```bash
mongo --host demo/localhost:27017,localhost:27018,localhost:27019 test
```

It will not work, and in the logs you will see:

```bash
MongoDB shell version v4.0.3
connecting to: mongodb://localhost:27017,localhost:27018,localhost:27019/test?replicaSet=demo
2019-03-30T13:34:37.975+0100 I NETWORK  [js] Starting new replica set monitor for demo/localhost:27017,localhost:27018,localhost:27019
2019-03-30T13:34:37.982+0100 I NETWORK  [js] Successfully connected to localhost:27018 (1 connections now open to localhost:27018 with a 5 second timeout)
2019-03-30T13:34:37.982+0100 I NETWORK  [ReplicaSetMonitor-TaskExecutor] Successfully connected to localhost:27019 (1 connections now open to localhost:27019 with a 5 second timeout)
2019-03-30T13:34:37.989+0100 I NETWORK  [ReplicaSetMonitor-TaskExecutor] Successfully connected to localhost:27017 (1 connections now open to localhost:27017 with a 5 second timeout)
2019-03-30T13:34:37.990+0100 I NETWORK  [ReplicaSetMonitor-TaskExecutor] changing hosts to demo/mongo-node1:27017,mongo-node2:27017,mongo-node3:27017 from demo/localhost:27017,localhost:27018,localhost:27019
2019-03-30T13:34:38.500+0100 W NETWORK  [js] Unable to reach primary for set demo
2019-03-30T13:34:38.500+0100 I NETWORK  [js] Cannot reach any nodes for set demo. Please check network connectivity and the status of the set. This has happened for 1 checks in a row.
```

We need to configure in the `/etc/hosts` file the container names in order for the connection to work when using `mongo` client or a Spring Boot application.

```bash
# mongodb cluster configuration
127.0.0.1 mongo-node1
127.0.0.1 mongo-node2
127.0.0.1 mongo-node3
```

After this is configured we can connect using:

```bash
mongo --host demo/localhost:27017,localhost:27018,localhost:27019 test
demo:PRIMARY> 
```

##### MongoDB GUI (Robo 3T)

```bash
brew cask install robo-3t
```

`ReplicatSet` connection configuration

<p><img src="/images/2019-03-26/DemoClusterMongoGUI.png" alt="DemoClusterMongoGUI" /></p>

##### Mongo client connection

```bash
brew install mongo
``` 

```bash
mongo --host demo/mongo-node1:27017,mongo-node2:27018,mongo-node3:27019 test
```

this uses the connection url:

```bash
mongodb://mongo-node1:27017,mongo-node2:27018,mongo-node3:27019/test?replicaSet=demo
``` 

##### Spring Data MongoDB connection

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://localhost:27017,localhost:27018,localhost:27019/test?replicaSet=demo
```

##### Useful MongoDB commands

```bash
show dbs
show collections
```

#### Useful docker commands

It is important not to allow a running container to consume too much of the host machine’s memory.
By default, each container’s access to the host machine’s CPU cycles is unlimited.

##### Limit a container’s access to memory

```bash
docker run -it --memory=1g ubuntu /bin/bash
```

```bash
docker stats

CONTAINER ID        NAME                CPU %               MEM USAGE / LIMIT     MEM %               NET I/O             BLOCK I/O           PIDS
cf04d3bd32b1        trusting_wu         0.00%               744KiB / 1GiB         0.07%               1.11kB / 0B         0B / 0B             1
```

##### CPU

Specify how much of the available CPU resources a container can use.
For instance, if the host machine has 4 CPUs (check with `nproc` command) and you set --cpus=1.5, the container is guaranteed at most one and a half of the CPUs.

```bash
docker run -it --cpus=1.5 ubuntu /bin/bash
```

#### Authentication

##### Create `admin` user

-- do this after creating the `ReplicaSet` without turning on authentication

```bash
use admin

admin = {
    user: "admin",
    pwd: "s3cr3t",
    roles: [ { role: "root", db:"admin" } ]
}

db.createUser(admin)

show users   // you need to in admin database
show roles

db.dropUser(<username>)

``` 

#### Turn on authentication

Add the following into your `mongod.config` configuration files for all 3 nodes

```bash
security:
  authorization: enabled
  keyFile: /etc/mongo/mongodb.key
```

The `keyFile` is needed for authentication between servers in the `ReplicaSet`, not for logging in. It can be created:

```bash
openssl rand -base64 1024 > mongodb.key
chmod 600 mongodb.key
```

#### Restart the cluster

```bash
docker container restart mongo-node1 mongo-node2 mongo-node3
```

#### Test login

```bash
mongo mongodb://admin:s3cr3t@localhost:27017/admin

show users

demo:PRIMARY> show users
{
	"_id" : "admin.admin",
	"user" : "admin",
	"db" : "admin",
	"roles" : [
		{
			"role" : "userAdminAnyDatabase",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"SCRAM-SHA-1",
		"SCRAM-SHA-256"
	]
}
```

Note that you can still connect to the cluster, with `mongo` only to the `admin` database you need authentication.

```bash
mongo 
use admin
show users

Error: command usersInfo requires authentication 

db.auth("admin","s3cr3t")
show users 

demo:PRIMARY> show users
{
	"_id" : "admin.admin",
	"user" : "admin",
	"db" : "admin",
	"roles" : [
		{
			"role" : "userAdminAnyDatabase",
			"db" : "admin"
		}
	],
	"mechanisms" : [
		"SCRAM-SHA-1",
		"SCRAM-SHA-256"
	]
}
```


##### Create additional users

```bash
use test
user =   {
           "user": "john",
           "pwd": "johnpass",
           "roles": [ { "role": "readWrite", db: "test" } ]
         }
         
db.createUser(user)         
``` 

##### Test connection

```bash
mongo mongodb://john:johnpass@localhost:27017/test

db.customer.find()   // should work
```

Note that with the `john` user you don't have access to `show users` and `show roles`, since we just gave `readWrite` role to the `john` user


##### Cleanup locally

```bash
docker container stop mongo-node1 mongo-node2 mongo-node3
docker container rm mongo-node1 mongo-node2 mongo-node3
```

#### Important

The client of a MongoDB replica set has to be able to connect to every member of the replica set by the host names listed in the replica set configuration.

