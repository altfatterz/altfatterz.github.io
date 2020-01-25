---
layout: post
title: GraphQL Server and Client
tags: [graphql, springboot]
---

[GraphQL](https://graphql.github.io/) is a query language for your API. It is also a specification which determines the validity schema on the server. 
It is strongly typed, the schema defines the GraphQL API's type system. It defines the possible objects that a client can access. A client can find information
about the schema via introspection. 

In this blog post we are going to develop a simple GraphQL server and client in Java. First we define the GraphQL schema
with the following root types. 

```graphql
type Query {
    bookById(id: ID!): Book
    books: [Book!]
}

type Mutation {
    addReview(input: AddReviewInput!): Boolean
}
```

We will be able to retrieve books by their id, retrieve all books and to add a review to a book. All types in GraphQL 
are nullable by default, with ! we can declare that the field is non nullable.

```graphql
type Book {
    id: ID!
    title: String!
    pageCount: Int
    publishedDate: Date
    author: Author!
    reviews: [Review]
}

type Author {
    firstName: String!
    lastName: String!
}

type Review {
    stars: Int!
    comment: String!
}

input AddReviewInput {
    bookId: ID!
    stars: Int!
    comment: String!
}

scalar Date

```

## Server 

For the GraphQL Server we use the Java implementation [https://www.graphql-java.com/](https://www.graphql-java.com/)
For the `bookById`, `books` and `addReview` fields we define a `DataFetcher`.
  
```kotlin
@Configuration
class GraphqlConfiguration(private val graphqlDataFetchers: GraphqlDataFetchers,
                           @Value("classpath:graphql/*.graphqls")
                           private val resources: Array<Resource>) {

    @Bean
    fun graphQL(): GraphQL? {
        return GraphQL.newGraphQL(buildGraphQLSchema()).build()
    }

    private fun buildGraphQLSchema(): GraphQLSchema {
        val schemaParser = SchemaParser()
        val typeDefinitionRegistry = TypeDefinitionRegistry()
        for (resource in resources) {
            typeDefinitionRegistry.merge(schemaParser.parse(resource.file))
        }
        val schemaGenerator = SchemaGenerator()
        return schemaGenerator.makeExecutableSchema(typeDefinitionRegistry, buildRuntimeWiring())
    }

    private fun buildRuntimeWiring(): RuntimeWiring {
        return RuntimeWiring.newRuntimeWiring()
                .scalar(ExtendedScalars.Date)
                .type(TypeRuntimeWiring.newTypeWiring("Query")
                        .dataFetcher("bookById", graphqlDataFetchers.bookByIdDataFetcher)
                        .dataFetcher("books", graphqlDataFetchers.books))
                .type(TypeRuntimeWiring.newTypeWiring("Mutation")
                        .dataFetcher("addReview", graphqlDataFetchers.addReview()))
                .build()
    }
}
```  

We can put a GraphQL API on any persistence layer. In this example we use [Spring Data JPA](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#reference).

```kotlin
@Component
class GraphqlDataFetchers(private val bookRepository: BookRepository,
                          private val reviewRepository: ReviewRepository) {

    val bookByIdDataFetcher: DataFetcher<Book?>
        get() = DataFetcher { env: DataFetchingEnvironment ->
            val id = env.getArgument<String>("id")
            bookRepository.findById(UUID.fromString(id))
        }
    
    ...
}
```

We define repositories for the following entities: `Book`, `Author` and `Review`.

```kotlin
interface BookRepository : Repository<Book, UUID> {
    fun save(book: Book): Book
    fun findById(id: UUID): Book?
    @EntityGraph(value = "books") fun findAll() : List<Book>
}

interface AuthorRepository : Repository<Author, UUID> {
    fun save(author: Author): Author
}

interface ReviewRepository : Repository<Review, UUID> {
    fun save(review: Review): Review
}
```

where the entities have the following definitions:

```kotlin
@Entity
@Table(name = "books")
@NamedEntityGraph(name = "books", attributeNodes = [NamedAttributeNode("author"),NamedAttributeNode("reviews")])
class Book(
        var title: String,
        @Column(name = "page_count") var pageCount: Int,
        @Column(name = "published_date") var publishedDate: LocalDate,
        @ManyToOne(fetch = FetchType.LAZY) var author: Author,
        @OneToMany @JoinColumn(name = "book_id") var reviews: MutableSet<Review> = HashSet(),
        @Id @GeneratedValue(generator = "UUID") @Type(type = "uuid-char") var id: UUID? = null) {
    fun addReview(review: Review) {
        reviews.add(review)
    }
}

@Entity
@Table(name = "authors")
class Author(
        @Column(name = "first_name")
        var firstName: String,
        @Column(name = "last_name")
        var lastName: String,
        @Id @GeneratedValue(generator = "UUID") @Type(type = "uuid-char") var id: UUID? = null)

@Entity
@Table(name = "reviews")
class Review(
        var stars: Int,
        var comment: String,
        @Id @GeneratedValue(generator = "UUID") @Type(type = "uuid-char") var id: UUID? = null
)
```

A simple GraphQL query 

```graphql
query {
  books {
    title  
  }
}
```

could return a response like

```json
{
  "data": {
    "books": [
      {
        "title": "The Da Vinci Code"
      },
      {
        "title": "Memoirs of a Geisha"
      },
      {
        "title": "The Alchemist"
      }
    ]
  }
}
```

Behind the scene it will fire the following SQL query to the database

```sql
Hibernate: 
    select
        book0_.id as id1_1_,
        book0_.author_id as author_i4_1_,
        book0_.page_count as page_cou2_1_,
        book0_.title as title3_1_ 
    from
        book book0_
```

The `@NamedEntityGraph` and `@EntityGraph` make sure to avoid the N+1 query problem. For the query

```graphql
query Books {
  books {
    title
    author{
      firstName
    }
    reviews {
      stars
      comment
    }
  }
}     
```

instead of having many queries on the `authors` and `reviews` tables we have only one query:

```sql
select
        book0_.id as id1_1_0_,
        author1_.id as id1_0_1_,
        reviews2_.id as id1_2_2_,
        book0_.author_id as author_i5_1_0_,
        book0_.page_count as page_cou2_1_0_,
        book0_.published_date as publishe3_1_0_,
        book0_.title as title4_1_0_,
        author1_.first_name as first_na2_0_1_,
        author1_.last_name as last_nam3_0_1_,
        reviews2_.comment as comment2_2_2_,
        reviews2_.stars as stars3_2_2_,
        reviews2_.book_id as book_id4_2_0__,
        reviews2_.id as id1_2_0__ 
    from
        books book0_ 
    left outer join
        authors author1_ 
            on book0_.author_id=author1_.id 
    left outer join
        reviews reviews2_ 
            on book0_.id=reviews2_.book_id
```

One more thing to note here that we didn't define a controller with `/graphql` endpoint. A `GraphQLController` is defined by using the following dependency.

```bash
dependencies {
    implementation("com.graphql-java:graphql-java-spring-boot-starter-webmvc:1.0")
    ...
}
```
 
## Client

With curl this is how we make a simple GraphQL request:

```bash
$ curl -i -X POST http://localhost:8080/graphql -H "Content-Type:application/json" -d '{"query":"query { books {title} } "}'
```

In GraphQL the request methods is always `POST`, wheather it is a query or a mutation.

With Java we can use Spring's RestTemplate to send a GraphQL POST request.

```java
ResponseEntity<String> response = restTemplate.exchange(graphqlEndpointUrl, HttpMethod.POST, request, String.class);
```

We can load the GraphQL query request from the classpath. A mutation query would look like this:

```graphql
mutation AddReview($input: AddReviewInput!) {
    addReview(input:$input)
}
```

where we externalised the `AddReviewInput` into a json file.

```json
[
  {
    "input": {
      "bookId": "b41c35c8-3b39-428d-a84b-97aebc7a1602",
      "stars": 5,
      "comment": "Its all about following your dream and about taking the risk of following your dreams"
    }
  },
  {
    "input": {
      "bookId": "c5ec9fb6-b586-449f-b869-bfe4057a4f0e",
      "stars": 5,
      "comment": "It is like eating fancy dessert at a gourmet restaurant"
    }
  }
]
``` 

We can build multipe GraphQL queries with the externalised inputs using the `variables` field   

```java
List<String> createJsonQueries(String graphql, String operationName, String variables) throws JsonProcessingException {
    ObjectNode wrapper = objectMapper.createObjectNode();
    wrapper.put("query", graphql);
    wrapper.put("operationName", operationName);
    List<String> queries = new ArrayList<>();
    JsonNode jsonNode = objectMapper.readTree(variables);
    for (int i = 0; i < jsonNode.size(); i++) {
        wrapper.set("variables", jsonNode.get(i));
        queries.add(objectMapper.writeValueAsString(wrapper));
    }
    return queries;
}
```

GraphQL is introspecitve. With the `__schema` query we can list all types in the schema

```graphql
query {
  __schema {
    types {
      name
      kind
      description
      fields {
        name
      }
    }
  }
}
```

With the `__type` query we can get details about a type

```graphql
query {
  __type(name: "Book") {
    name
    kind
    description
    fields {
      name
    }
  }
}
``` 

[`GraphiQL`](https://github.com/graphql/graphiql) is a great tool to explore a GraphQL API. Normally you include a pre-bundled version into your GraphQL server. 
Then based on the schema definition which it gets from the GraphQL Server and you can start playing with the API.
Here is live demo [https://graphql.github.io/swapi-graphql/](https://graphql.github.io/swapi-graphql/)

[`GraphQL Playground`](https://github.com/prisma-labs/graphql-playground) is another popular GraphQL tool. It can be used even 
as a desktop app. It has query history, configuration of HTTP headers and tabs. 
![graphql-playground](/images/2020-01-25/graphql-playground.png)

## Conclusion

In this blog post we saw how easy to create a simple GraphQL server anc client. 
The code examples can be found in this repository [https://github.com/altfatterz/graphql-demos](https://github.com/altfatterz/graphql-demos)