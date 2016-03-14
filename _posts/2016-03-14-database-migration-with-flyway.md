---
layout: post
title: Database migration with Flyway
tags: [flywaydb, postgresql, springboot, docker]
---

In this post I describe how to evolve a database schema using [Flyway](https://flywaydb.org/). I will create a simple service which manages athletes using postgresql as datastore.

The initial database schema: 

```sql
CREATE TABLE athletes (
    id BIGSERIAL,
    first_name varchar(255) not null,
    last_name varchar(255) not null,
    PRIMARY KEY (id)
);
```

Now let's imagine a business requirement to store also the country of origin of the athletes and to be able to search athletes based on their first name. This requires, beside the necessary code changes, updates to the database schema. I need to add a new `country` field and I need to create an index on the `firstName` field.
I would like to roll out the database schema changes together with the code changes as an atomic unit.
Let's see how can I do it with Flyway.

I have set up the athletes service using postgresql with docker-compose, the code is available on my [github](https://github.com/altfatterz/spring-boot-flyway) account.

```
version: '2'
services:
  postgres:
    image: athletes-datastore-image
    container_name: athletes-datastore
    build: build/db
    ports:
      - "5432:5432"
  web:
    image: athletes-service-image
    container_name: athletes-service
    build: build/libs
    depends_on:
      - postgres # athletes-datastore will be started before the athletes-service
    ports:
      - "8080:8080"
    links:
      - postgres
```

When the `athletes-datastore` container is created it creates the initial database schema seen above with some example data.
 
```
FROM postgres:9.5.1
MAINTAINER altfatterz@gmail.com
ENV POSTGRES_USER docker
ENV POSTGRES_DB mydb
COPY 1_schema.sql /docker-entrypoint-initdb.d/
COPY 2_data.sql /docker-entrypoint-initdb.d/
```

When the `athletes-service` container starts up I will use flyway to apply the necessary database migrations.
  
```
spring.datasource.url=jdbc:postgresql://postgres/mydb
spring.datasource.username=docker

# fail application start if schema is not present
spring.jpa.hibernate.ddl-auto=validate 

# disable database initialisation with Spring JDBC
spring.datasource.initialize=false

flyway.enabled=true

# Controls weather to a automatically call baseline when migrate is executed against a non-empty schema with no metadata table.
# Only migrations above the baseLineVersion (default 1) will be applied
flyway.baseline-on-migrate=true
```  
  
The database schema changes are encapsulated in the `db/migration` folder 
  
V2__Add_country_field_to_athletes_table.sql:

```sql
ALTER TABLE athletes ADD COLUMN country VARCHAR(200);
```

V3__Create_index_first_name_in_athletes_table.sql:

```sql
CREATE INDEX first_name_idx ON athletes (first_name);
```

Let's build the project and start up the services 

```
$ ./gradlew clean build
$ docker-compose up
```  

In the logs it is visible that the database changes are applied

```
o.f.core.internal.util.VersionPrinter    : Flyway 4.0 by Boxfuse
o.f.c.i.dbsupport.DbSupportFactory       : Database: jdbc:postgresql://postgres/mydb (PostgreSQL 9.5)
o.f.core.internal.command.DbValidate     : Successfully validated 2 migrations (execution time 00:00.024s)
o.f.c.i.metadatatable.MetaDataTableImpl  : Creating Metadata table: "public"."schema_version"
o.f.core.internal.command.DbBaseline     : Successfully baselined schema with version: 1
o.f.core.internal.command.DbMigrate      : Current version of schema "public": 1
o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version 2 - Add country field to athletes table
o.f.core.internal.command.DbMigrate      : Migrating schema "public" to version 3 - Create index first name in athletes table
o.f.core.internal.command.DbMigrate      : Successfully applied 2 migrations to schema "public" (execution time 00:00.047s)
```

And indeed if I connect to the postgresql instance via

```
psql -h 192.168.99.100 -d mydb -U docker
```

the `schema_version` records the applied changes:

```
installed_rank  | version |                description                |   type   |                      script                       |  checksum  | installed_by |        installed_on        | execution_time | success
----------------+---------+-------------------------------------------+----------+---------------------------------------------------+------------+--------------+----------------------------+----------------+---------
              1 | 1       | << Flyway Baseline >>                     | BASELINE | << Flyway Baseline >>                             |            | docker       | 2016-03-14 09:24:38.979852 |              0 | t
              2 | 2       | Add country field to athletes table       | SQL      | V2__Add_country_field_to_athletes_table.sql       | -674532233 | docker       | 2016-03-14 09:24:39.043319 |              4 | t
              3 | 3       | Create index first name in athletes table | SQL      | V3__Create_index_first_name_in_athletes_table.sql | 1143920342 | docker       | 2016-03-14 09:24:39.064954 |              4 | t
```

If I stop the docker containers with `docker-compose stop` (I needed to disconnect from psql otherwise stopping `athletes-datastore` blocks) and start it again via `docker-compose start` in the logs it is visible that the schema changes are up to date.
 
```
o.f.core.internal.util.VersionPrinter    : Flyway 4.0 by Boxfuse
o.f.c.i.dbsupport.DbSupportFactory       : Database: jdbc:postgresql://postgres/mydb (PostgreSQL 9.5)
o.f.core.internal.command.DbValidate     : Successfully validated 3 migrations (execution time 00:00.020s)
o.f.core.internal.command.DbMigrate      : Current version of schema "public": 3
o.f.core.internal.command.DbMigrate      : Schema "public" is up to date. No migration necessary.
``` 

Set country of origin of an athlete:

```
$ echo '{"country":"Jamaica"}' | http patch http://192.168.99.100:8080/athletes/1
```

Search based on first name:

```
$ http get http://192.168.99.100:8080/athletes/search/findByFirstNameIgnoreCase/\?firstName\=usain

HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Mon, 14 Mar 2016 10:43:29 GMT
Server: Apache-Coyote/1.1
Transfer-Encoding: chunked
X-Application-Context: application:postgresql

{
    "_embedded": {
        "athletes": [
            {
                "_links": {
                    "athlete": {
                        "href": "http://192.168.99.100:8080/athletes/1"
                    },
                    "self": {
                        "href": "http://192.168.99.100:8080/athletes/1"
                    }
                },
                "country": "Jamaica",
                "firstName": "Usain",
                "lastName": "Bolt"
            }
        ]
    },
    "_links": {
        "self": {
            "href": "http://192.168.99.100:8080/athletes/search/findByFirstNameIgnoreCase/?firstName=usain"
        }
    }
}
```

To easily stop containers and remove the containers, networks, volumes and images created by `docker-compose up` I use  

```
docker-compose down -v --rmi=all
```

I think Flyway is a very useful tool for handling database schema changes especially in a continuous delivery pipeline. 



