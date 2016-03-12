---
layout: post
title: Dockerized Spring Boot service on AWS
tags: [docker, springboot, awscloud]
---

In this post I describe how to deploy a simple dockerized Spring Boot service to [AWS](https://aws.amazon.com). The example service provides a RESTful API to manage schools and leverages mongodb for storage. 

```java
@SpringBootApplication
public class Application {

	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}

@Document(collection = "schools")
class School {

	@Id
	private String id;

	private final String name;
	private final String city;

	@JsonCreator
	public School(@JsonProperty("name") String name, @JsonProperty("city") String city) {
		this.name = name;
		this.city = city;
	}

	public String getName() {
		return name;
	}
	public String getCity() {
		return city;
	}
}

@RepositoryRestResource(collectionResourceRel = "schools", path = "/schools")
interface SchoolRepository extends MongoRepository<School, String> {

	List<School> findByCityIgnoreCase(@Param("city") String city);
}
```

The following `Dockerfile` is used to create the image:  

```
FROM java:8
MAINTAINER altfatterz@gmail.com
VOLUME /tmp
ADD spring-boot-docker-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
RUN bash -c 'touch /app.jar'
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

```

Using [Docker Compose](https://docs.docker.com/compose/#installation-and-set-up) and the following `docker-compose.yml` configuration you can easily test the application.  

```
version: '2'
services:
  mongodb:
    container_name: schools-datastore
    image: mongo:3.2
    command: mongod --smallfiles
  web:
    container_name: schools-service
    build: build/libs
    image: school-service-image
    depends_on: # schools-datastore will be started before the schools-service
      - mongodb
    ports:
      - "8080:8080"
    links:
      - mongodb
    environment:
      SPRING_DATA_MONGODB_URI: mongodb://mongodb/test
```

by running 

```
$ docker-compose up
```

This command will create and start two containers one for mongodb and one for the school service. 

```
$ docker ps -a

CONTAINER ID   IMAGE                 COMMAND                 PORTS                   NAMES
f49bdf199461   school-service-image  "java -Djava.security"  0.0.0.0:8080->8080/tcp  schools-service
573f330dc378   mongo:3.2             "/entrypoint.sh mongo"  27017/tcp               schools-datastore
```

Let's test it. Note, your docker host might have a different IP, check it with `echo $DOCKER_HOST` command.  

```
$ echo '{"name":"St. Bonifatiuscollege ", "city":"Utrecht"}' | http post http://192.168.99.100:8080/schools

HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Sat, 12 Mar 2016 12:09:20 GMT
Location: http://192.168.99.100:8080/schools/56e406f0e4b0e760889f4559
Server: Apache-Coyote/1.1
Transfer-Encoding: chunked
X-Application-Context: application

{
    "_links": {
        "school": {
            "href": "http://192.168.99.100:8080/schools/56e406f0e4b0e760889f4559"
        },
        "self": {
            "href": "http://192.168.99.100:8080/schools/56e406f0e4b0e760889f4559"
        }
    },
    "city": "Utrecht",
    "name": "St. Bonifatiuscollege "
}
```

I am using [httpie](https://github.com/jkbrzt/httpie) which is more user friendly `curl` and comes with syntax highlighting. 

And searching:

```
$ http get http://192.168.99.100:8080/schools/search/findByCityIgnoreCase\?city\=utrecht

HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Sat, 12 Mar 2016 12:17:35 GMT
Server: Apache-Coyote/1.1
Transfer-Encoding: chunked
X-Application-Context: application

{
    "_embedded": {
        "schools": [
            {
                "_links": {
                    "school": {
                        "href": "http://192.168.99.100:8080/schools/56e406f0e4b0e760889f4559"
                    },
                    "self": {
                        "href": "http://192.168.99.100:8080/schools/56e406f0e4b0e760889f4559"
                    }
                },
                "city": "Utrecht",
                "name": "St. Bonifatiuscollege "
            }
        ]
    },
    "_links": {
        "self": {
            "href": "http://192.168.99.100:8080/schools/search/findByCityIgnoreCase?city=utrecht"
        }
    }
}
```

Now let's deploy this simple service to AWS.

First I need to push the image to a registry. For this example I am using [DockerHub](https://hub.docker.com/) but [AWS ECR](https://aws.amazon.com/ecr/) would have been also a good choice.

```
$ cd spring-boot-docker
$ ./gradlew clean build
$ docker build -t altfatterz/spring-boot-docker-demo:0.0.1 build/libs
$ docker push altfatterz/spring-boot-docker-demo:0.0.1
```
![DockerHub](/images/dockerhub.png)

I start a `t2.micro` instance of `Amazon Linux AMI` image with a security group allowing incomming SSH and HTTP requests from any source (0.0.0.0/0).
After successfully connecting to the instance I install Docker on it.
 
```
$ sudo yum update -y
$ sudo yum install -y docker
$ sudo service docker start
```

For convenience I add the `ec2-user` to the `docker` group in order to execute Docker commands without `sudo`.

```
$ sudo usermod -a -G docker ec2-user
$ exit
```

I need to logout and login again in order for the settings to take effect. With `docker info` I verify that the information about the Docker installation is returned successfully without using `sudo`.
Now I am ready to start my `altfatterz/spring-boot-docker-demo:0.0.1` image.

```
$ docker run -d -p 80:8080 \
  -e SPRING_DATA_MONGODB_URI=mongodb://test:test@ds019478.mlab.com:19478/zoltan \ 
  altfatterz/spring-boot-docker-demo:0.0.1 --name schools-service
```

I link the 8080 port on the Docker container to the 80 port on the EC2 instance with the `-p` option.
Using the `-e` option I create an environment variable connecting a mongodb service provided by [mLab](https://mlab.com/). mLab lets me to create a free sandbox mongodb service up to 500 MB which is great for prototyping. 
After making sure the docker container starts successfully by checking the logs via `docker logs -f <container-id>` I can finally test it:

```
$ curl http://localhost                     // from the EC2 instance
$ http get http://<ec2-instance-public-ip>  // from my host
```

I have now a running Docker container on AWS using my example image. The problem is though that it is really a manual process and does not scale well if I need to deploy many containers, routing the traffic to the containers (load balancing), making sure the containers are running (monitoring).
I need more automation. In another blog post I will describe how you can deploy this simple service using [AWS ECS](http://aws.amazon.com/ecs/)

The example service can be found on my [github](https://github.com/altfatterz/spring-boot-docker) account.




