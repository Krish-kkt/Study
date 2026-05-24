# Spring Boot — SDE 2 Interview Topics

> **How to use this file:** This is your master checklist. Each section maps to a depth of knowledge expected at the SDE 2 level — not just "what is it" but "how does it work, why does it exist, and what can go wrong."

---

## 1. Core Spring Fundamentals (Must Know Cold)

These are foundational — interviewers will build every follow-up on top of these.

### 1.1 IoC (Inversion of Control) & Dependency Injection
- What IoC means and why it exists (loose coupling, testability)
- Three types of DI: **Constructor**, **Setter**, **Field** injection
- Why constructor injection is preferred over `@Autowired` on a field
- What happens if Spring can't resolve a dependency at startup

### 1.2 The Spring Container
- `BeanFactory` vs `ApplicationContext` — difference and when each matters
- How `ApplicationContext` loads and initializes beans
- `AnnotationConfigApplicationContext` vs `WebApplicationContext`

### 1.3 Bean Lifecycle
- Full lifecycle: Instantiation → Populate Properties → `BeanNameAware` → `BeanFactoryAware` → `@PostConstruct` → Ready → `@PreDestroy` → Destroy
- `InitializingBean` / `DisposableBean` vs `@PostConstruct` / `@PreDestroy`
- `BeanPostProcessor` — what it is, why Spring Security and AOP use it heavily

### 1.4 Bean Scopes
- `singleton` (default) — one instance per container
- `prototype` — new instance on every `getBean()` call
- `request`, `session`, `application` — web scopes
- **Gotcha:** injecting a `prototype`-scoped bean into a `singleton` — it won't behave as prototype. Fix: `@Lookup` or `ObjectProvider`

### 1.5 Stereotype Annotations
- `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`
- These are all aliases for `@Component` with added semantics
- `@Repository` adds exception translation (converts JDBC exceptions to Spring's `DataAccessException`)

---

## 2. Spring Boot Fundamentals

### 2.1 What Spring Boot Adds Over Spring
- **Auto-configuration:** Spring Boot reads your classpath and auto-configures beans you'd otherwise write manually
- **Starter POMs:** curated dependency bundles (`spring-boot-starter-web`, `spring-boot-starter-data-jpa`, etc.)
- **Embedded server:** Tomcat/Jetty/Undertow baked in — no WAR deployment needed
- **Production-ready features:** Actuator, health checks, metrics out of the box

### 2.2 `@SpringBootApplication`
- Composite of three annotations:
  - `@Configuration` — this class declares beans
  - `@EnableAutoConfiguration` — trigger auto-config
  - `@ComponentScan` — scan this package and subpackages for components
- The main class should sit at the root package so `@ComponentScan` covers everything

### 2.3 Auto-Configuration Deep Dive
- Mechanism: `spring.factories` (pre-Boot 3) / `AutoConfiguration.imports` (Boot 3+)
- `@Conditional` annotations: `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
- How to **disable** a specific auto-config: `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})`
- How to **debug** what was auto-configured: `--debug` flag prints auto-config report

### 2.4 Configuration & Externalized Config
- Priority order (highest to lowest): CLI args → `SPRING_APPLICATION_JSON` → env variables → `application-{profile}.yml` → `application.yml`
- `@Value("${property.name}")` for simple values
- `@ConfigurationProperties(prefix = "app")` for binding a group of properties to a POJO — preferred for complex config
- `@Validated` on `@ConfigurationProperties` to fail fast on bad config at startup

### 2.5 Profiles
- `@Profile("prod")` on a bean — only registered when that profile is active
- `spring.profiles.active=prod` in env or `application.yml`
- Common pattern: `application.yml` (shared) + `application-dev.yml` + `application-prod.yml`

---

## 3. Web Layer (REST APIs)

### 3.1 Controller Basics
- `@RestController` = `@Controller` + `@ResponseBody`
- `@RequestMapping` at class level sets base path; method-level annotations refine it
- `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, `@PatchMapping`

### 3.2 Request Binding
- `@PathVariable` — extract from URL path (`/users/{id}`)
- `@RequestParam` — extract from query string (`/users?page=1`)
- `@RequestBody` — deserialize JSON body into a Java object (uses Jackson)
- `@RequestHeader` — extract a specific HTTP header

### 3.3 `ResponseEntity`
- Full control over HTTP response: body, status code, headers
- Prefer `ResponseEntity<T>` over just returning `T` when status code matters
- Common pattern: `ResponseEntity.ok(body)`, `ResponseEntity.notFound().build()`, `ResponseEntity.status(HttpStatus.CREATED).body(dto)`

### 3.4 Global Exception Handling
- `@ControllerAdvice` + `@ExceptionHandler` — intercepts exceptions from any controller
- Return a consistent error response DTO with status, message, timestamp
- `@ResponseStatus(HttpStatus.NOT_FOUND)` as a shortcut on custom exceptions
- Spring's `DefaultHandlerExceptionResolver` already handles some exceptions (e.g., `MethodArgumentNotValidException`)

### 3.5 Validation
- `@Valid` or `@Validated` on `@RequestBody` parameter to trigger Bean Validation
- Constraint annotations: `@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Email`, `@Pattern`
- Handle `MethodArgumentNotValidException` in `@ControllerAdvice` to return field-level errors
- Custom validators: implement `ConstraintValidator<A, T>`

### 3.6 Filters vs Interceptors vs AOP
| | Filter | Interceptor | AOP |
|---|---|---|---|
| Layer | Servlet (before Spring) | Spring MVC (after DispatcherServlet) | Any Spring bean method |
| Use for | Auth tokens, CORS, logging all requests | Pre/post processing of controller calls | Cross-cutting concerns (logging, transactions) |
| Registered via | `FilterRegistrationBean` or `@Component` | `WebMvcConfigurer.addInterceptors()` | `@Aspect` |

### 3.7 `DispatcherServlet`
- The front controller — all requests go through it
- Routes to the right controller via `HandlerMapping`
- Know this flow: `Request → Filter Chain → DispatcherServlet → HandlerMapping → HandlerAdapter → Controller → ViewResolver/MessageConverter → Response`

---

## 4. Data Layer

### 4.1 Spring Data JPA Overview
- JPA is the specification; Hibernate is the default implementation
- `EntityManager` is the core JPA API — Spring wraps it so you rarely use it directly
- Spring Data repositories give you CRUD for free: `JpaRepository<Entity, ID>`

### 4.2 Repository Hierarchy
```
Repository (marker)
  └── CrudRepository (save, findById, delete, findAll)
        └── PagingAndSortingRepository (findAll with Pageable)
              └── JpaRepository (flush, saveAndFlush, deleteInBatch, getOne)
```

### 4.3 Query Methods
- Derived queries from method names: `findByEmailAndActiveTrue(String email)`
- `@Query("SELECT u FROM User u WHERE u.email = :email")` — JPQL
- `@Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)` — SQL
- `@Modifying` + `@Transactional` for UPDATE/DELETE queries

### 4.4 JPA Entity Mapping
- `@Entity`, `@Table(name = "users")`
- `@Id`, `@GeneratedValue(strategy = GenerationType.IDENTITY)`
- `@Column(nullable = false, unique = true, length = 255)`
- `@CreationTimestamp`, `@UpdateTimestamp` (Hibernate-specific)

### 4.5 Relationships
- `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`
- `mappedBy` — the non-owning side; the owning side has the foreign key column
- `CascadeType`: `ALL`, `PERSIST`, `MERGE`, `REMOVE` — be careful with `REMOVE` on collections
- `FetchType.LAZY` vs `FetchType.EAGER`
  - `@OneToMany` default: LAZY
  - `@ManyToOne` default: EAGER (often change this to LAZY)
  - EAGER on large associations causes performance disasters

### 4.6 N+1 Problem — Critical Interview Topic
- **Problem:** Load 100 orders → each order lazily loads its user → 101 queries total
- **Fix 1:** `@EntityGraph` to join-fetch associations in one query
- **Fix 2:** `@Query("SELECT o FROM Order o JOIN FETCH o.user")`
- **Fix 3:** `@BatchSize(size = 20)` on the collection — batches lazy loads
- Enable SQL logging (`spring.jpa.show-sql=true`) to spot this in code review

### 4.7 Transactions
- `@Transactional` defaults: propagation `REQUIRED`, isolation `DEFAULT` (DB default), rollback on `RuntimeException` only
- **Propagation types:**
  - `REQUIRED` — join existing or create new
  - `REQUIRES_NEW` — always create new, suspend existing
  - `NESTED` — savepoint within existing transaction
  - `SUPPORTS` — use existing if present, no tx otherwise
- **Isolation levels:** READ_UNCOMMITTED < READ_COMMITTED < REPEATABLE_READ < SERIALIZABLE
- **Self-invocation problem:** calling a `@Transactional` method from within the same class bypasses the proxy — the transaction annotation has no effect
- Rollback: `@Transactional(rollbackFor = CheckedException.class)` to rollback on checked exceptions

### 4.8 Connection Pooling (HikariCP)
- Spring Boot defaults to HikariCP — the fastest JDBC connection pool
- Key properties: `maximum-pool-size` (default 10), `minimum-idle`, `connection-timeout`, `idle-timeout`
- Pool exhaustion = your entire service hangs waiting for a connection; tune pool size carefully

---

## 5. Caching

### 5.1 Spring Cache Abstraction
- `@EnableCaching` on a `@Configuration` class
- `@Cacheable("users")` — cache return value; on hit, skip method body
- `@CachePut("users")` — always run method, update cache with result
- `@CacheEvict("users")` — remove from cache; `allEntries = true` clears entire cache
- Cache key: by default uses method parameters; customize with `key = "#id"`

### 5.2 Cache Providers
- Default: `ConcurrentHashMap` (in-memory, no eviction, not distributed)
- Production: **Redis** (distributed, TTL support) via `spring-boot-starter-data-redis`
- **Caffeine** — fast in-process cache with eviction policies

### 5.3 Cache Pitfalls
- Caching `null` — disable with `unless = "#result == null"`
- Stale data — always have a TTL or explicit eviction strategy
- Caching mutable objects — can cause subtle bugs if caller mutates the cached object

---

## 6. AOP (Aspect-Oriented Programming)

### 6.1 Core Concepts
- **Aspect:** class containing cross-cutting logic
- **Advice:** the action (what runs): `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around`
- **Pointcut:** expression that selects which methods to intercept
- **Join point:** the actual method execution being intercepted

### 6.2 Proxy Mechanism
- Spring AOP is proxy-based — Spring wraps your bean in a proxy at startup
- Two proxy types: JDK dynamic proxy (interface-based) or CGLIB (subclass-based)
- **Same-class method calls bypass the proxy** — this is why `@Transactional` self-invocation fails

### 6.3 Common Use Cases
- Logging method entry/exit and execution time
- Security checks before method execution
- Transaction management (Spring's `@Transactional` is itself AOP)
- Retry logic

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();
        log.info("{} took {}ms", pjp.getSignature(), System.currentTimeMillis() - start);
        return result;
    }
}
```

---

## 7. Asynchronous & Scheduling

### 7.1 `@Async`
- `@EnableAsync` on a `@Configuration` class
- `@Async` on a method — runs in a separate thread pool
- Return type: `void` or `CompletableFuture<T>` (for result retrieval)
- **Gotcha:** same-class invocation bypasses the proxy — async won't fire
- Configure a custom `ThreadPoolTaskExecutor` bean; don't use the default single-thread pool in production

### 7.2 `@Scheduled`
- `@EnableScheduling` on a `@Configuration` class
- `@Scheduled(fixedRate = 5000)` — every 5s from start of last execution
- `@Scheduled(fixedDelay = 5000)` — 5s after last execution finishes
- `@Scheduled(cron = "0 0 * * * ?")` — cron expression (every hour)
- Runs in a single thread by default — long tasks block subsequent scheduled tasks

---

## 8. Testing

### 8.1 Test Slices (Know These Cold)
| Annotation | Loads | Use for |
|---|---|---|
| `@SpringBootTest` | Full application context | Integration tests |
| `@WebMvcTest` | Web layer only (no DB) | Controller unit tests |
| `@DataJpaTest` | JPA layer + H2 in-memory | Repository tests |
| `@RestClientTest` | HTTP client beans | Testing REST clients |

### 8.2 `@WebMvcTest`
- Only loads controllers, filters, `ControllerAdvice` — no service/repo beans
- Use `@MockBean` to stub out services
- Use `MockMvc` to fire test requests

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean UserService userService;

    @Test
    void getUser_returnsUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new UserDto(1L, "Alice"));
        mockMvc.perform(get("/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("Alice"));
    }
}
```

### 8.3 `@DataJpaTest`
- Rolls back each test by default — no test pollution
- Uses an in-memory H2 DB by default; use `@AutoConfigureTestDatabase(replace = NONE)` for real DB

### 8.4 Mockito Essentials
- `@Mock` — plain Mockito mock (no Spring context)
- `@MockBean` — mock registered in Spring context (replaces the real bean)
- `@Spy` / `@SpyBean` — real object but with selective stubbing
- `@InjectMocks` — inject mocks into the subject under test
- `verify(mock, times(1)).method(arg)` — assert interaction happened

### 8.5 Test Best Practices
- Test behavior, not implementation — don't over-specify interactions
- Use `@ExtendWith(MockitoExtension.class)` for plain unit tests
- Avoid `@SpringBootTest` for unit tests — too slow; use it only for integration tests

---

## 9. Spring Boot Actuator

### 9.1 Key Endpoints
| Endpoint | What it shows |
|---|---|
| `/actuator/health` | App health (UP/DOWN), DB, disk, custom checks |
| `/actuator/info` | App metadata (version, git commit) |
| `/actuator/metrics` | JVM, HTTP, DB metrics |
| `/actuator/env` | All resolved properties |
| `/actuator/beans` | All beans in context |
| `/actuator/threaddump` | Current thread states |
| `/actuator/httptrace` | Last N HTTP requests |

### 9.2 Custom Health Indicator
```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        boolean up = checkGateway();
        return up ? Health.up().build() : Health.down().withDetail("reason", "timeout").build();
    }
}
```

### 9.3 Security Consideration
- Expose only `health` and `info` publicly; secure all others or expose only over management port
- `management.endpoints.web.exposure.include=health,info`

---

## 10. Microservice-Relevant Spring Boot Topics

### 10.1 `RestTemplate` vs `WebClient`
- `RestTemplate`: synchronous, blocking — deprecated in Spring 6 / Boot 3, still widely used
- `WebClient`: non-blocking, reactive, part of Spring WebFlux — preferred for new code
- For simple sync use, `RestTemplate` is fine; in high-throughput services, prefer `WebClient`

### 10.2 Resilience4j (Circuit Breaker)
- `@CircuitBreaker(name = "service", fallbackMethod = "fallback")` — opens circuit after failure threshold
- `@Retry(name = "service")` — retries on failure with backoff
- `@RateLimiter`, `@Bulkhead` — protect against overload
- States: CLOSED (normal) → OPEN (rejecting calls) → HALF_OPEN (testing recovery)

### 10.3 Distributed Tracing
- Spring Boot 3 uses **Micrometer Tracing** (previously Spring Cloud Sleuth)
- Adds `traceId` and `spanId` to MDC — appears in logs automatically with Logback
- Export traces to Zipkin or Jaeger

### 10.4 Service-to-Service Communication Patterns
- Synchronous: REST (RestTemplate/WebClient), gRPC
- Asynchronous: Kafka, RabbitMQ (Spring AMQP)
- OpenFeign (`@FeignClient`) — declarative HTTP client, generates implementation from interface

---

## 11. Performance & Common Pitfalls

### 11.1 Lazy Loading in Non-Transaction Context
- Accessing a lazy-loaded collection outside a transaction throws `LazyInitializationException`
- Fix: Open Session in View (OSIV) — Spring Boot enables this by default, but it hides N+1 problems
- Better fix: Use DTOs and project only what you need; never expose entities directly to the web layer

### 11.2 Entity vs DTO
- Never return JPA entities from controllers — it leaks schema, causes serialization issues with lazy proxies, and couples API to DB schema
- Use DTOs (Data Transfer Objects) for all API responses/requests
- MapStruct or manual mapping for entity ↔ DTO conversion

### 11.3 Transaction Pitfalls
1. **Self-invocation** — calling `@Transactional` method from same class → no transaction
2. **Checked exceptions don't rollback** — must set `rollbackFor`
3. **Too-wide transactions** — don't put HTTP calls or heavy computation inside `@Transactional`
4. **Swallowing exceptions** — catching exceptions inside a `@Transactional` method prevents rollback

### 11.4 Common Memory Leaks / Startup Issues
- Circular dependency — Spring can detect constructor injection circular deps at startup
- Bean not found — check component scan base packages
- Multiple beans of same type — use `@Primary` or `@Qualifier`

---

## 12. Spring Boot 3.x / Modern Changes (SDE 2 Should Know)

- Requires Java 17+ minimum
- Jakarta EE 9+ — package renamed from `javax.*` to `jakarta.*` (e.g., `javax.persistence` → `jakarta.persistence`)
- `spring.factories` replaced by `AutoConfiguration.imports`
- Native image support via GraalVM (AOT compilation)
- Problem Details for HTTP APIs (`application/problem+json`) — RFC 7807 support built in
- Virtual threads support (Java 21 + Spring Boot 3.2): `spring.threads.virtual.enabled=true`

---

## 13. Interview Question Patterns

These are the question shapes that appear most at SDE 2 level. Prepare a structured answer for each.

### "Explain how X works internally"
- Auto-configuration, `@Transactional`, AOP proxies, bean lifecycle, `@Async`

### "What happens when..."
- App starts up (component scan → auto-config → bean init → embedded server)
- A request comes in (filter chain → DispatcherServlet → controller → response)
- A `@Transactional` method throws an exception

### "What's wrong with this code?"
- N+1 query from eager/lazy misuse
- `@Transactional` self-invocation
- Prototype bean injected into singleton
- Caching a mutable object
- Async method called within same class

### "How would you design/improve X?"
- Rate limiting on REST endpoints → Resilience4j / Bucket4j
- Reduce DB load → caching with Redis + `@Cacheable`
- Background job → `@Scheduled` or Kafka consumer
- High-throughput API → WebClient + virtual threads

---

## Study Priority (Focus Order for Interview Prep)

| Priority | Topic | Why |
|---|---|---|
| 1 | Bean lifecycle, DI, scopes | Foundation of every follow-up |
| 2 | `@Transactional` deep-dive | Most common real-world bug topic |
| 3 | N+1 problem + fixes | Always asked in data layer |
| 4 | Exception handling (`@ControllerAdvice`) | Practical API design question |
| 5 | `@WebMvcTest` + Mockito | Testing is expected at SDE 2 |
| 6 | Auto-configuration mechanism | Shows depth beyond basic usage |
| 7 | AOP proxy mechanism | Ties together transactions, security, async |
| 8 | Caching with Redis | Cloud/microservice-scale question |
| 9 | Actuator + health checks | Production readiness question |
| 10 | Resilience4j / circuit breaker | Microservice architecture question |
