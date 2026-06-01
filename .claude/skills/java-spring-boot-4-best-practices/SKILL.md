---
name: java-spring-boot-4-best-practices
description: This skill provides guidance for building applications with Spring Boot 4.
---

# Spring Boot 4

This skill provides guidance for building applications with Spring Boot 4.

## Jackson 3 (JSON Processing)

Spring Boot 4 uses Jackson 3, which has significant changes from Jackson 2.

### Package Changes

Jackson 3 uses the `tools.jackson` package instead of `com.fasterxml.jackson`:

```java
// Jackson 3 (Spring Boot 4)
import tools.jackson.databind.JsonMapper;
import tools.jackson.databind.ObjectReader;
import tools.jackson.databind.ObjectWriter;
import tools.jackson.core.JacksonException;

// NOT these (Jackson 2)
// import com.fasterxml.jackson.databind.ObjectMapper;
// import com.fasterxml.jackson.core.JsonProcessingException;
```

### Use JsonMapper Instead of ObjectMapper

Spring Boot 4 auto-configures a `JsonMapper` bean. Inject it directly:

```java
@Service
public class MyService {

    private final JsonMapper jsonMapper;

    public MyService(JsonMapper jsonMapper) {
        this.jsonMapper = jsonMapper;
    }

    public String toJson(Object obj) throws JacksonException {
        return jsonMapper.writeValueAsString(obj);
    }

    public <T> T fromJson(String json, Class<T> type) throws JacksonException {
        return jsonMapper.readValue(json, type);
    }
}
```

### Exception Handling

Jackson 3 uses `JacksonException` (unchecked) instead of `JsonProcessingException` (checked):

```java
// Jackson 3
import tools.jackson.core.JacksonException;

try {
    String json = jsonMapper.writeValueAsString(obj);
} catch (JacksonException e) {
    // handle error
}
```

### Common Annotations

Annotations remain the same but use the new package:

```java
import tools.jackson.annotation.JsonProperty;
import tools.jackson.annotation.JsonIgnore;
import tools.jackson.annotation.JsonFormat;

public record Person(
    @JsonProperty("full_name") String name,
    @JsonIgnore String internalId
) {}
```

### Creating a Custom JsonMapper

If you need a custom configuration:

```java
@Configuration
public class JacksonConfig {

    @Bean
    public JsonMapper jsonMapper() {
        return JsonMapper.builder()
            .findAndAddModules()
            .build();
    }
}
```

## Null Safety with JSpecify

Spring Framework 7 (used by Spring Boot 4) adopts JSpecify annotations for null safety, enabling compile-time null checks and better IDE support.

### Annotations

JSpecify provides two key annotations from `org.jspecify.annotations`:

- **`@NullMarked`** - Declares an entire package as null-safe by default (all types are non-null unless specified)
- **`@Nullable`** - Marks specific fields, parameters, or return types that can be null

### Package-Level Null Safety

Create a `package-info.java` file to mark an entire package as null-safe:

```java
// package-info.java
@NullMarked
package com.example.myapp;

import org.jspecify.annotations.NullMarked;
```

### Using @Nullable for Exceptions

With `@NullMarked` applied, use `@Nullable` to indicate where nulls are permitted:

```java
import org.jspecify.annotations.Nullable;

@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // Return type is non-null by default (enforced by @NullMarked)
    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    // Explicitly mark nullable parameters
    public List<User> search(@Nullable String name) {
        if (name == null) {
            return userRepository.findAll();
        }
        return userRepository.findByNameContaining(name);
    }
}
```

### Benefits

- **Compile-time safety**: Static analysis tools catch null issues before runtime
- **IDE support**: Better autocomplete and null warnings
- **Kotlin interoperability**: Seamless integration with Kotlin's null safety system

## RestTestClient (Testing)

Spring Framework 7 introduces `RestTestClient`, a unified testing API that replaces the need to switch between `MockMvc`, `WebTestClient`, and `TestRestTemplate`.

### Binding Methods

RestTestClient offers five binding methods for different testing scenarios:

| Method | Use Case | Speed |
|--------|----------|-------|
| `bindToController` | Unit tests without Spring context | Fast |
| `bindToMockMvc` | Tests with Spring MVC validation/security | Medium |
| `bindToApplicationContext` | Full app with real services | Medium |
| `bindToServer` | End-to-end with HTTP layer | Slow |
| `bindToRouterFunction` | Functional/reactive endpoints | Fast |

### Unit Test with bindToController

Test controllers in isolation using mocks:

```java
@ExtendWith(MockitoExtension.class)
class TodoControllerTest {

    @Mock
    private TodoService todoService;

    private RestTestClient client;

    @BeforeEach
    void setUp() {
        client = RestTestClient.bindToController(new TodoController(todoService)).build();
    }

    @Test
    void shouldGetAllTodos() {
        when(todoService.findAll()).thenReturn(List.of(
            new Todo(1L, "Learn Spring", false)
        ));

        client.get()
            .uri("/api/todos")
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$[0].title").isEqualTo("Learn Spring");
    }
}
```

### Integration Test with bindToMockMvc

Test with Spring MVC features like validation and security:

```java
@WebMvcTest(TodoController.class)
class TodoControllerMvcTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private TodoService todoService;

    @Test
    void shouldValidateInput() {
        RestTestClient client = RestTestClient.bindToMockMvc(mockMvc).build();

        client.post()
            .uri("/api/todos")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new Todo(null, "", false))
            .exchange()
            .expectStatus().isBadRequest();
    }
}
```

### End-to-End Test with bindToServer

Test against a running server:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class TodoIntegrationTest {

    @LocalServerPort
    private int port;

    @Test
    void shouldCreateAndRetrieveTodo() {
        RestTestClient client = RestTestClient.bindToServer()
            .baseUrl("http://localhost:" + port)
            .build();

        client.post()
            .uri("/api/todos")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(new Todo(null, "Integration test", false))
            .exchange()
            .expectStatus().isCreated();
    }
}
```

### Common Assertions

```java
// Status assertions
.expectStatus().isOk()
.expectStatus().isCreated()
.expectStatus().isBadRequest()
.expectStatus().isNotFound()

// Body assertions with JSONPath
.expectBody()
.jsonPath("$.name").isEqualTo("value")
.jsonPath("$[0].id").exists()
.jsonPath("$.items.length()").isEqualTo(3)
```

## Resilience (Retry & Concurrency)

Spring Framework 7 includes built-in resilience capabilities, eliminating the need for external libraries like Spring Retry.

### Enable Resilient Methods

First, enable the resilience features:

```java
@Configuration
@EnableResilientMethods
public class ResilienceConfig {
}
```

### @Retryable

Automatically retry failed method calls with exponential backoff:

```java
@Service
public class ExternalApiService {

    @Retryable(
        maxAttempts = 4,
        delay = 500,
        multiplier = 2.0,  // 500ms, 1s, 2s, 4s
        maxDelay = 5000,
        jitter = 100       // randomness to prevent thundering herd
    )
    public String fetchData(String id) {
        // Method that might fail - will retry with increasing delays
        return restClient.get()
            .uri("/api/data/{id}", id)
            .retrieve()
            .body(String.class);
    }
}
```

### @ConcurrencyLimit

Control concurrent method execution to prevent resource exhaustion:

```java
@Service
public class HeavyProcessingService {

    @ConcurrencyLimit(2)
    public String performHeavyOperation(String taskId) {
        // Only 2 concurrent executions allowed
        // Additional requests queue and wait
        return processTask(taskId);
    }
}
```

### Combining Annotations

Use both annotations together for robust resilience:

```java
@ConcurrencyLimit(1)
@Retryable(maxAttempts = 2, delay = 1000)
public String criticalOperation(String operationId) {
    // Single-threaded execution with automatic retry
    return performCriticalWork(operationId);
}
```

### @Retryable Parameters

| Parameter | Description |
|-----------|-------------|
| `maxAttempts` | Maximum number of retry attempts |
| `delay` | Initial delay in milliseconds |
| `multiplier` | Multiplier for exponential backoff |
| `maxDelay` | Maximum delay cap in milliseconds |
| `jitter` | Random variance to prevent synchronized retries |

## Modular Auto-Configuration

Spring Boot 4 introduces modular auto-configuration where features previously bundled together are now split into separate, focused modules. This is a **breaking change** from Spring Boot 3.x.

### Why Modular?

- **Smaller footprint**: Include only what you need
- **Clearer dependencies**: Explicit about what your app uses
- **Better optimization**: Ideal for microservices and cloud deployments

### Quick Migration (Classic Module)

For quick migration from Spring Boot 3.x, add the classic module to restore previous behavior:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-classic</artifactId>
</dependency>
```

### Modular Approach (Recommended)

For new projects or gradual migration, add only the modules you need:

```xml
<!-- REST Client Support -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-restclient</artifactId>
</dependency>

<!-- JPA/Hibernate Support -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-data-jpa</artifactId>
</dependency>

<!-- Security Support -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure-security</artifactId>
</dependency>
```

### Common Modules

| Module | Purpose |
|--------|---------|
| `spring-boot-autoconfigure-classic` | All Spring Boot 3.x auto-configurations |
| `spring-boot-starter-restclient` | REST client support |
| `spring-boot-autoconfigure-data-jpa` | JPA/Hibernate support |
| `spring-boot-autoconfigure-security` | Spring Security support |

### Migration Strategy

1. **Immediate**: Add `spring-boot-autoconfigure-classic` to maintain existing behavior
2. **Gradual**: Replace classic with specific modules as you identify what your app actually uses
3. **New projects**: Start with only the modules you need

## HTTP Interface Clients

Spring Boot 4 introduces `@ImportHttpServices` for zero-boilerplate declarative HTTP clients.

### Define the Interface

Use `@HttpExchange` to declare your HTTP client:

```java
@HttpExchange(url = "https://jsonplaceholder.typicode.com", accept = "application/json")
public interface TodoService {

    @GetExchange("/todos")
    List<Todo> getAllTodos();

    @GetExchange("/todos/{id}")
    Todo getTodoById(@PathVariable Long id);

    @PostExchange("/todos")
    Todo createTodo(@RequestBody Todo todo);
}
```

### Register with @ImportHttpServices

In Spring Boot 4, a single annotation handles all the wiring:

```java
@Configuration(proxyBeanMethods = false)
@ImportHttpServices(TodoService.class)
public class HttpClientConfig {
    // That's it!
}
```

### Old Way (Spring Boot 3.x)

Previously required manual factory setup:

```java
// ❌ No longer needed in Spring Boot 4
@Bean
public TodoService todoService(RestClient.Builder restClientBuilder) {
    var restClient = restClientBuilder
        .baseUrl("https://jsonplaceholder.typicode.com")
        .build();

    var adapter = RestClientAdapter.create(restClient);
    var factory = HttpServiceProxyFactory.builderFor(adapter).build();
    return factory.createClient(TodoService.class);
}
```

### Multiple Clients

Register multiple HTTP interfaces in one annotation:

```java
@Configuration(proxyBeanMethods = false)
@ImportHttpServices({TodoService.class, UserService.class, PostService.class})
public class HttpClientConfig {
}
```

### Common Annotations

| Annotation | Purpose |
|------------|---------|
| `@HttpExchange` | Base URL and content type for the interface |
| `@GetExchange` | HTTP GET request |
| `@PostExchange` | HTTP POST request |
| `@PutExchange` | HTTP PUT request |
| `@DeleteExchange` | HTTP DELETE request |
| `@PatchExchange` | HTTP PATCH request |
