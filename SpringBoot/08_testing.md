# 08 — Testing

---

## 8.1 Testing Strategy Overview

A healthy test suite uses multiple layers of tests:

```
         /\
        /  \    E2E Tests (few, slow, full system)
       /────\
      / Integ \  Integration Tests (@SpringBootTest)
     /──────────\
    / Unit Tests  \  (many, fast, isolated — @WebMvcTest, @DataJpaTest, plain JUnit)
   /______________\
```

The goal at SDE 2 level: write fast unit tests for business logic, integration tests for the glue (controllers, repos), and avoid a heavy reliance on slow full-context tests.

---

## 8.2 Test Slices — Know These Cold

Spring Boot provides "slices" that load only part of the application context, making tests much faster than full `@SpringBootTest`:

| Annotation | What it loads | What it doesn't load |
|---|---|---|
| `@WebMvcTest` | Controllers, filters, `@ControllerAdvice`, Jackson | Services, repositories, full context |
| `@DataJpaTest` | JPA repositories, entity manager, H2 in-memory | Web layer, services |
| `@JsonTest` | Jackson serialization/deserialization only | Everything else |
| `@RestClientTest` | `RestTemplate`/WebClient beans | Full context |
| `@SpringBootTest` | Entire application context | Nothing — everything loads |

---

## 8.3 `@WebMvcTest` — Controller Tests

Test your controller in isolation. Services are mocked.

```java
@WebMvcTest(UserController.class)  // loads only UserController + web layer
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;  // simulates HTTP — no actual server started

    @MockBean  // registers a Mockito mock in the Spring context
    private UserService userService;

    @Autowired
    private ObjectMapper objectMapper;

    @Test
    void getUser_whenUserExists_returns200() throws Exception {
        // Arrange
        UserResponse expected = new UserResponse(1L, "Alice", "alice@example.com");
        when(userService.findById(1L)).thenReturn(expected);

        // Act + Assert
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Alice"))
                .andExpect(jsonPath("$.email").value("alice@example.com"));
    }

    @Test
    void getUser_whenNotFound_returns404() throws Exception {
        when(userService.findById(99L)).thenThrow(new UserNotFoundException(99L));

        mockMvc.perform(get("/api/users/99"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.message").value("User not found with id: 99"));
    }

    @Test
    void createUser_withValidBody_returns201() throws Exception {
        CreateUserRequest request = new CreateUserRequest("Bob", "bob@example.com", 25);
        UserResponse created = new UserResponse(2L, "Bob", "bob@example.com");
        when(userService.create(any(CreateUserRequest.class))).thenReturn(created);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(2));
    }

    @Test
    void createUser_withBlankName_returns400WithFieldError() throws Exception {
        CreateUserRequest invalid = new CreateUserRequest("", "bob@example.com", 25);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalid)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.fieldErrors.name").exists());
    }
}
```

---

## 8.4 `@DataJpaTest` — Repository Tests

Test repository queries against a real (in-memory) database:

```java
@DataJpaTest  // loads only JPA layer + H2 in-memory DB
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;  // for setup/teardown in tests

    @Test
    void findByEmail_whenExists_returnsUser() {
        // Arrange — persist test data
        User user = new User("Alice", "alice@example.com", UserStatus.ACTIVE);
        entityManager.persistAndFlush(user);

        // Act
        Optional<User> found = userRepository.findByEmail("alice@example.com");

        // Assert
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Alice");
    }

    @Test
    void findByStatus_returnOnlyActiveUsers() {
        entityManager.persist(new User("Active User", "a@test.com", UserStatus.ACTIVE));
        entityManager.persist(new User("Inactive User", "b@test.com", UserStatus.INACTIVE));
        entityManager.flush();

        List<User> active = userRepository.findByStatus(UserStatus.ACTIVE);

        assertThat(active).hasSize(1);
        assertThat(active.get(0).getName()).isEqualTo("Active User");
    }

    @Test
    void existsByEmail_whenExists_returnsTrue() {
        entityManager.persistAndFlush(new User("Alice", "alice@test.com", UserStatus.ACTIVE));
        assertThat(userRepository.existsByEmail("alice@test.com")).isTrue();
        assertThat(userRepository.existsByEmail("nobody@test.com")).isFalse();
    }
}
```

### Testing with Real Database

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)  // use real DB
@Transactional(propagation = Propagation.NOT_SUPPORTED)  // no auto-rollback
class UserRepositoryIntegrationTest { ... }
```

---

## 8.5 `@SpringBootTest` — Full Integration Tests

Loads the complete application context. Slow — use sparingly:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserApiIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;  // real HTTP client

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    void createAndFetchUser_endToEnd() {
        CreateUserRequest request = new CreateUserRequest("Alice", "alice@test.com", 25);

        ResponseEntity<UserResponse> created = restTemplate.postForEntity(
            "/api/users", request, UserResponse.class);

        assertThat(created.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(created.getBody()).isNotNull();

        Long id = created.getBody().getId();
        ResponseEntity<UserResponse> fetched = restTemplate.getForEntity(
            "/api/users/" + id, UserResponse.class);

        assertThat(fetched.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(fetched.getBody().getName()).isEqualTo("Alice");
    }
}
```

### `WebEnvironment` Options

| Option | What happens | Use for |
|---|---|---|
| `RANDOM_PORT` | Starts embedded server on random port | Full E2E with real HTTP |
| `DEFINED_PORT` | Uses server.port (default 8080) | When port matters |
| `MOCK` (default) | No server; uses MockMvc | Integration test without real HTTP |
| `NONE` | No web environment | CLI apps, service layer tests |

---

## 8.6 Mockito Essentials

### Mock vs Spy vs Stub

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock                         // complete mock — all methods return default (null/0/false)
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks                  // creates UserService and injects mocks
    private UserService userService;

    @Test
    void createUser_savesAndSendsEmail() {
        // Stubbing — define what the mock returns
        User savedUser = new User(1L, "Alice", "alice@example.com");
        when(userRepository.save(any(User.class))).thenReturn(savedUser);

        // Act
        UserResponse result = userService.create(new CreateUserRequest("Alice", "alice@example.com", 25));

        // Assert result
        assertThat(result.getId()).isEqualTo(1L);

        // Verify interactions
        verify(userRepository, times(1)).save(any(User.class));
        verify(emailService, times(1)).sendWelcome("alice@example.com", "Alice");
        verifyNoMoreInteractions(emailService);
    }

    @Test
    void createUser_whenEmailExists_throwsException() {
        when(userRepository.existsByEmail("alice@example.com")).thenReturn(true);

        assertThatThrownBy(() ->
            userService.create(new CreateUserRequest("Alice", "alice@example.com", 25)))
            .isInstanceOf(DuplicateEmailException.class)
            .hasMessage("Email already exists: alice@example.com");

        verify(userRepository, never()).save(any());  // save should not be called
    }
}
```

### `@Spy` — Partial Mock

```java
@Spy  // real object — methods execute for real, but you can stub specific ones
private AuditService auditService = new AuditService();

@Test
void test() {
    doNothing().when(auditService).log(any());  // stub only this method
    // all other methods run for real
}
```

### `@MockBean` vs `@Mock`

```java
// @Mock — plain Mockito, no Spring context needed
@Mock
private UserRepository userRepository;  // use in plain unit tests with @ExtendWith(MockitoExtension)

// @MockBean — registers mock IN the Spring context, replaces the real bean
// Use in @WebMvcTest or @SpringBootTest
@MockBean
private UserService userService;  // Spring injects this mock into controllers
```

### ArgumentCaptor — Verify What Was Passed

```java
@Test
void createUser_emailContainsName() {
    ArgumentCaptor<EmailMessage> captor = ArgumentCaptor.forClass(EmailMessage.class);
    when(userRepository.save(any())).thenReturn(new User(1L, "Alice", "alice@test.com"));

    userService.create(new CreateUserRequest("Alice", "alice@test.com", 25));

    verify(emailService).send(captor.capture());
    EmailMessage sentEmail = captor.getValue();
    assertThat(sentEmail.getBody()).contains("Alice");
    assertThat(sentEmail.getTo()).isEqualTo("alice@test.com");
}
```

---

## 8.7 AssertJ — Fluent Assertions

Prefer AssertJ over JUnit's `assertEquals` — more readable and better error messages:

```java
// JUnit style — hard to read failure messages
assertEquals(expected, actual);

// AssertJ — readable, great failure messages
assertThat(actual).isEqualTo(expected);
assertThat(list).hasSize(3).contains("Alice").doesNotContain("Bob");
assertThat(optional).isPresent().hasValue("something");
assertThat(string).startsWith("Hello").endsWith("World").containsIgnoringCase("ello");
assertThat(number).isGreaterThan(0).isLessThanOrEqualTo(100);

// Exception assertions
assertThatThrownBy(() -> service.doThing())
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessage("invalid input")
    .hasCauseInstanceOf(NullPointerException.class);

// Soft assertions — collect all failures instead of stopping at first
SoftAssertions.assertSoftly(softly -> {
    softly.assertThat(user.getName()).isEqualTo("Alice");
    softly.assertThat(user.getEmail()).isEqualTo("alice@test.com");
    softly.assertThat(user.getAge()).isGreaterThan(0);
});
```

---

## 8.8 Testing Best Practices

### Name Tests Clearly

```java
// Bad
@Test void test1() { ... }

// Good — method_condition_expectedOutcome
@Test void findById_whenUserExists_returnsUser() { ... }
@Test void findById_whenUserNotFound_throwsNotFoundException() { ... }
@Test void createUser_withDuplicateEmail_throwsDuplicateEmailException() { ... }
```

### Use `@Nested` for Organization

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Nested
    class GetUser {
        @Test void whenExists_returns200() { ... }
        @Test void whenNotFound_returns404() { ... }
    }

    @Nested
    class CreateUser {
        @Test void withValidBody_returns201() { ... }
        @Test void withBlankName_returns400() { ... }
        @Test void withInvalidEmail_returns400() { ... }
    }
}
```

### Avoid Testing Implementation Details

```java
// Bad — tests internal behavior, breaks when you refactor
verify(userRepository, times(1)).save(any());
verify(cacheService, times(1)).put("user:1", user);

// Good — tests observable behavior
assertThat(result.getId()).isNotNull();
assertThat(userRepository.findById(savedId)).isPresent();
```

---

## Tricky Interview Questions

**Q: What's the difference between `@Mock` and `@MockBean`?**

- `@Mock` (Mockito): pure Mockito mock, no Spring context. Use with `@ExtendWith(MockitoExtension.class)` for fast unit tests.
- `@MockBean` (Spring Boot Test): registers the mock in the Spring application context, replacing the real bean. Use in slice tests (`@WebMvcTest`) or `@SpringBootTest` when the mock needs to be injected into a Spring-managed class.

```java
// @Mock — no Spring, fast
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock UserRepository repo;
    @InjectMocks UserService service;
}

// @MockBean — needs Spring context
@WebMvcTest(UserController.class)
class UserControllerTest {
    @MockBean UserService service;  // Spring injects this into UserController
}
```

---

**Q: `@DataJpaTest` automatically rolls back each test. How do you disable this?**

```java
@DataJpaTest
@Transactional(propagation = Propagation.NOT_SUPPORTED)  // disables auto-rollback
class RepoTest { ... }

// Or per method:
@Test
@Rollback(false)
void persistentTest() { ... }
```

---

**Q: You write a `@SpringBootTest` test and it fails because the actual database isn't available. How do you handle this?**

Options:
1. Use an in-memory DB: `@AutoConfigureTestDatabase` (H2 by default)
2. Use Testcontainers — spins up a real Docker container for the test:

```java
@SpringBootTest
@Testcontainers
class OrderIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

**Q: How do you test a method annotated with `@Async`?**

Either disable async for the test, or wait for the result:

```java
// Option 1: Override async config for tests (synchronous execution)
@TestConfiguration
public class TestAsyncConfig implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        return new SyncTaskExecutor();  // runs in same thread synchronously
    }
}

// Option 2: Use CompletableFuture and wait
CompletableFuture<Report> future = reportService.generateReport(1L);
Report result = future.get(5, TimeUnit.SECONDS);
assertThat(result).isNotNull();

// Option 3: Use Awaitility for async assertions
await().atMost(5, TimeUnit.SECONDS)
       .until(() -> notificationRepository.existsByUserId(userId));
```
