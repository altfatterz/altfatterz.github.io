---
layout: post
title: Trying out Spring Boot 1.4.0 new features and enhancements
tags: [springboot]
---

Spring Boot 1.4 M2 upgrades to Spring 4.3.0-RC1 and defaults to Hibernate 5.1

# Spring 4.3.0-RC1

In Spring 4.3.0-RC1 there are lots of new features and enhancements. The full list you can find it [here](http://docs.spring.io/spring/docs/4.3.0.RC1/spring-framework-reference/htmlsingle/#new-in-4.3).

Some of my favourites are

1.  The introduction of composed annotations from `@RequestMapping` like for example `@GetMapping` as a shortcut for `@RequestMapping(method = RequestMethod.GET)`

    ```java
    @RestController
    class ChuckNorrisFactController {

        @GetMapping("/")
        public ChuckNorrisFact getOneRandomly() {
            return service.getOneRandomly();
        }
    }
    ```

2.  No need to specify `@Autowired` for constructor injection if the target bean defines only one constructor which is mostly the case.

    ```java
    @RestController
    class ChuckNorrisFactController {

        private ChuckNorrisFactService service;

        public ChuckNorrisFactController(ChuckNorrisFactService service) {
            this.service = service;
        }
    }
    ```

3. The `SpringRunner` alias for the `SpringJUnit4ClassRunner`

# Hibernate 5

The general upgrade instruction from the Hibernate team can be found [here](https://github.com/hibernate/hibernate-orm/blob/5.0/migration-guide.adoc). If you have upgrade problems switching to Hibernate 5 can be postponed at a later stage by setting the `hibernate.version` in your `pom.xml`. Note though that the Spring team will not support Hibernate 4.3 from Spring Boot 1.4.

# Spring Boot 1.4.0-M2

Let's see what new features and enhancements Spring Boot 1.4.0-M2 contains on top of the above. The full list you can find in [`M1`](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-1.4.0-M1-Release-Notes) and [`M2`](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-1.4.0-M2-Release-Notes)  release notes.

Here I selected couple of my favourites.

## Testing support

The biggest change comes regarding testing support. There are two new modules [`spring-boot-test`](https://github.com/spring-projects/spring-boot/tree/v1.4.0.M2/spring-boot-test) and [`spring-boot-test-autoconfigure`](https://github.com/spring-projects/spring-boot/tree/v1.4.0.M2/spring-boot-test-autoconfigure)

### @SpringBootTest

Spring already comes with great support for writing integration tests leveraging the [`spring-test`](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/integration-testing.html) module. The only problem might be that it uses lots of annotations (`@SpringApplicationConfiguration`, `@ContextConfiguration`, `@IntegrationTest`, `@WebIntegrationTest`) to accomplish it.

Spring Boot 1.4 tries to simplify this by providing a single `@SpringBootTest` annotation.

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Sql("/init-for-full-integration-test.sql")
public class ApplicationTests {

	@Autowired
	private TestRestTemplate template;

	@Test
	public void random() {
		ResponseEntity<String> fact = template.getForEntity("/", String.class); // cannot use ChuckNorrisFact deserialization

		assertThat(fact.getBody(), containsString("Chuck Norris sleeps with a pillow under his gun."));
	}
}
```

Here as you can see for convenience the injected `TestRestTemplate` is already configured to make calls against the started server, no need to inject the port with `@Value("${local.server.port}")`. However if you would like to know the port there is a better `@LocalServerPort` annotation.

### Testing slices of the application

Sometimes you would like to test a simple "slice" of the application instead of auto-configuring the whole application. Spring Boot 1.4 introduces 3 new test annotations:

1. `@WebMvcTest` - for testing the controller layer
2. `@JsonTest` - for testing the JSON marshalling and unmarshalling
3. `@DataJpaTest` - for testing the repository layer

#### @WebMvcTest

In order to test only the controller layer or often a single controller the `@WebMvcTest` used in combination with @MockBean can be handy. @Service or @Repository components will not be scanned.

```java
@RunWith(SpringRunner.class)
@WebMvcTest(ChuckNorrisFactController.class)
public class CheckNorrisFactControllerTests {

    @Autowired
    private MockMvc mvc;

    @MockBean
    private ChuckNorrisFactService chuckNorrisFactService;

    @Test
    public void getOneRandomly() throws Exception {
        given(chuckNorrisFactService.getOneRandomly()).willReturn(new ChuckNorrisFact("Chuck Norris counted to infinity twice."));

        mvc.perform(get("/").accept(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(content().json("{'fact':'Chuck Norris counted to infinity twice.'}"));
    }
}
```

#### @JsonTest

To test JSON marshalling and unmarshalling you can use the `@JsonTest` annotation. `@JsonTest` will auto-configure Jackson ObjectMappers, @JsonComponent beans (see later).

```java
@RunWith(SpringRunner.class)
@JsonTest
public class ChuckNorrisFactJsonTests {

    private JacksonTester<ChuckNorrisFact> json;

    @Test
    public void serialize() throws IOException {
        ChuckNorrisFact fact = new ChuckNorrisFact("When Chuck Norris turned 18, his parents moved out.");
        JsonContent<ChuckNorrisFact> write = this.json.write(fact);

        assertThat(this.json.write(fact)).isEqualToJson("expected.json");
    }

    @Test
    public void deserialize() throws IOException {
        String content = "{\"fact\":\"Chuck Norris knows Victoria's secret.\"}";

        assertThat(this.json.parse(content)).isEqualTo(new ChuckNorrisFact("Chuck Norris knows Victoria's secret."));
    }
}
```

#### @DataJpaTest

To test the JPA layer of an application you can use @DataJpaTest. By default it configures an in-memory database, scans for @Entity classes and configures the Spring Data JPA repositories. @Service, @Controller, @RestController beans will not be loaded.

```java
@RunWith(SpringRunner.class)
@DataJpaTest
public class ChuckNorrisFactRepositoryTests {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private ChuckNorrisFactRepository repository;

    @Test
    public void getOneRandomly() {
        String fact = "If at first you don't succeed, you're not Chuck Norris.";
        entityManager.persist(new ChuckNorrisFact(fact));

        List<ChuckNorrisFact> facts = repository.getOneRandomly(new PageRequest(0, 10));
        assertThat(facts.get(0).getText(), is(fact));
    }
}
```

### Datasource binding

In Spring Boot 1.4 via `spring.datasource` property namespace only the common datasource properties are set. For datasource provider specific ones new namespaces were introduced.
For example if you are using [HikariCP](https://github.com/brettwooldridge/HikariCP) as a datasource provider you can set

```
spring.datasource.hikari.maximum-pool-size=5
spring.datasource.hikari.connection-timeout=10

```

In IntelliJ with configuration assistance is nice to check which properties can be set for the given datasource provider instead of having them all mixed under `spring.datasource` namespace.

### @JsonComponent

With `@JsonComponent` you can easier register custom JSON serializers and deserializers. Below yon can see the additional utility classes (`JsonObjectSerializer` and `JsonObjectDeserializer`) which simplify even more the serialization and deserialization logic.

```java
@JsonComponent
public class ChuckNorrisFactJson {

    public static class Serializer extends JsonObjectSerializer<ChuckNorrisFact> {

        @Override
        protected void serializeObject(ChuckNorrisFact value, JsonGenerator jgen, SerializerProvider provider) throws IOException {
            jgen.writeStringField("fact", value.getText());
        }
    }

    public static class Deserilizer extends JsonObjectDeserializer<ChuckNorrisFact> {

        @Override
        protected ChuckNorrisFact deserializeObject(JsonParser jsonParser, DeserializationContext context, ObjectCodec codec, JsonNode tree) throws IOException {
            String fact = tree.get("fact").asText();
            return new ChuckNorrisFact(fact);
        }
    }
}
```

### Image Banners

You can use an image to render the ASCII banner. For inspiration have a look what people already created [tweetbootbanner](https://twitter.com/hashtag/tweetabootbanner)


## Summary

If you want to try out these new features and enhancements have a look to this [github](https://github.com/altfatterz/chuck-norris-facts) repository. Also have a look at [Phil Webb](https://twitter.com/phillip_webb)'s excellent [post](https://spring.io/blog/2016/04/15/testing-improvements-in-spring-boot-1-4) about testing support improvements in Spring Boot 1.4





