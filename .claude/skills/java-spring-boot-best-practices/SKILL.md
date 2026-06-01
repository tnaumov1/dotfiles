---
name: java-spring-boot-best-practices
description: Spring Boot development patterns and anti-patterns for building clean, maintainable applications. Use when writing Spring Boot code, reviewing code, creating controllers, services, repositories, tests, or configuration. Covers dependency injection, REST API design, exception handling, testing strategies, and configuration management.
---

# Spring Boot Best Practices

Apply these patterns when working on any Spring Boot project.

## Dependency Injection

### Prefer constructor injection, never use field injection 

Prefer constructor injection, never use field injection or set injection unless you're in a test context

```java
// ❌ Don't
@RestController
public class UserController {
    @Autowired
    private UserService userService;
}

// ✅ Do
@RestController
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

Why: Constructor injection enables immutability (final fields), simplifies testing (no Spring context needed), exposes circular dependencies at compile time, and makes dependencies explicit.

## REST Controller Design

### Use `@RestController` for REST controllers

Never use `@Controller` with `@ResponseBody` to write a REST controller.

### Let HTTP methods convey intent, not URIs

```java
// ❌ Don't
@GetMapping("/getUser/{id}")
@PostMapping("/createUser")
@PostMapping("/deleteUser/{id}")

// ✅ Do
@GetMapping("/users/{id}")
@PostMapping("/users")
@DeleteMapping("/users/{id}")
```

### Return appropriate status codes

Always specify the status code in the `@ResponseStatus` annotation for happy cases.

For any bad cases throw an Exception and handle it in ExceptionHanlder, but never handle exception in controller
```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public User createUser(@RequestBody User user) {
    return userService.save(user);
}

@GetMapping("/{id}")
@ResponseStatus(HttpStatus.OK)
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

### Use DTOs for request/response objects

For every domain model create a DTO (Data Transfer Object) that represents the data that needs to be transferred over the wire

Include in DTO only the data that is needed for the client to consume.

Add a validation annotation to the Request DTO to ensure that the client sends the correct data

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public UserCreateResponse createUser(@RequestBody @Validated UserCreateRequest request) {
    return userService.save(user);
}

public record UserCreateRequest(@NotBlank String email, @PositiveOrZero Integer age) {
}
public record UserCreateResponse(Long id, String email) {
}
```

## Data Transfer Objects

### Use records for DTOs

Name record accordingly to their usage (Request, Response or Update)

if data transfer object is used in multiple places you can call it like Dto

```java
public record UserDto(String email, String name) {}
public record CreateUserRequest(String email, String name) {}
public record UserResponse(Long id, String email, String name) {}
```

Records are immutable, concise, and work well with Jackson’s serialization.

### Use Jakarta Validation

Use Jakarta Validation constraints annotations to declare the validation that should be performed at each field

```java
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.PositiveOrZero;

public record UserCreateRequest(@NotBlank String email, @PositiveOrZero Integer age) {
}
```

## Exception Handling

**Use `@RestControllerAdvice` with specific exception handlers:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException e) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, e.getMessage());
        problem.setTitle("Resource Not Found");
        return problem;
    }

    @ExceptionHandler(ValidationException.class)
    public ProblemDetail handleValidation(ValidationException e) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.BAD_REQUEST, e.getMessage());
        problem.setTitle("Validation Error");
        return problem;
    }
}
```

Use RFC 7807 `ProblemDetail` (Spring 6+) for standardized error responses. Avoid catching broad `Exception.class`.

## Testing

**Use test slices instead of `@SpringBootTest` for focused tests:**

| Annotation | Purpose |
|------------|---------|
| `@WebMvcTest` | Controller layer (web layer only) |
| `@DataJpaTest` | Repository layer (JPA only) |
| `@JsonTest` | JSON serialization |
| `@WebFluxTest` | Reactive controllers |
| `@RestClientTest` | REST client testing |

```java
// ❌ Don't - loads entire context
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest { }

// ✅ Do - loads only web layer
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;
}
```

**For unit tests, use plain JUnit + Mockito without Spring:**

```java
class UserServiceTest {
    private UserRepository userRepository;
    private UserService userService;

    @BeforeEach
    void setUp() {
        userRepository = mock(UserRepository.class);
        userService = new UserService(userRepository);
    }
}
```

This is only possible when using constructor injection.

## Configuration

**Use `@ConfigurationProperties` instead of scattered `@Value` annotations:**

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
    @DefaultValue("smtp.example.com") String host,
    @DefaultValue("587") int port,
    @DefaultValue("false") boolean ssl
) {}
```

Benefits: type safety, validation at startup, IDE autocomplete in YAML/properties files, grouped configuration.

**Never hardcode environment-specific values.**

## Package Structure

**Prefer package-by-feature over package-by-layer:**

```
// ❌ Package by layer (requires public everywhere)
com.example.controllers/
com.example.services/
com.example.repositories/

// ✅ Package by feature (enables package-private)
com.example.user/
    UserController.java
    UserService.java
    UserRepository.java
com.example.order/
    OrderController.java
    OrderService.java
```

Package-by-feature keeps related code together and enables proper encapsulation with package-private visibility.

## Interface Overuse

**Don't create interfaces for single implementations:**

```java
// ❌ Don't - unnecessary abstraction
public interface UserService { }
public class UserServiceImpl implements UserService { }

// ✅ Do - interface when you have multiple implementations or need it for testing/proxying
public class UserService { }
```

Create interfaces when you have multiple implementations or explicit architectural reasons, not by default.

## HTTP Clients

**Use declarative HTTP Interface Clients (Spring 6+):**

```java
@HttpExchange("/api/users")
public interface UserClient {
    @GetExchange("/{id}")
    User findById(@PathVariable Long id);

    @PostExchange
    User create(@RequestBody User user);
}
```

For simpler cases, use `RestClient` over `RestTemplate`.

## Observability

**Use Micrometer Observation API for consistent metrics and tracing:**

```java
@RestController
public class UserController {
    private final ObservationRegistry registry;

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return Observation.createNotStarted("user.fetch", registry)
            .observe(() -> userService.findById(id));
    }
}
```
