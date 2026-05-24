# 11 — Performance & Common Pitfalls

---

## 11.1 Entity vs DTO — A Core Design Principle

**Never expose JPA entities directly as API responses.** This is one of the most common mistakes in Spring Boot apps.

### Why Entities Are Dangerous as API Responses

```java
// BAD — returning entity directly
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).orElseThrow();
}
```

Problems:
1. **Lazy loading + serialization = `LazyInitializationException`**: Jackson tries to serialize lazy-loaded collections → throws exception (unless OSIV is enabled, which causes N+1)
2. **Exposes internal schema**: DB column names become API contract; a DB rename breaks API consumers
3. **Over-exposure**: passwords, internal flags, timestamps leak to clients
4. **Circular references**: `User` has `List<Order>`, `Order` has `User` → infinite loop during serialization
5. **Tight coupling**: API contract is tied to DB structure

### The Right Way — Always Use DTOs

```java
// The entity (internal, DB-mapped)
@Entity
@Table(name = "users")
public class User {
    @Id private Long id;
    private String name;
    private String email;
    private String passwordHash;       // should NEVER leave the server
    private UserStatus status;
    @OneToMany(mappedBy = "user")
    private List<Order> orders;        // lazy-loaded
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// The response DTO (external, API contract)
public record UserResponse(
    Long id,
    String name,
    String email,
    String status
    // no password, no orders unless explicitly requested, no internal timestamps
) {}

// The request DTO (input validation boundary)
public record CreateUserRequest(
    @NotBlank String name,
    @Email @NotBlank String email,
    @NotBlank String password,
    @Min(18) int age
) {}
```

### Manual Mapping vs MapStruct

```java
// Manual mapping — fine for simple cases
@Component
public class UserMapper {
    public UserResponse toResponse(User user) {
        return new UserResponse(user.getId(), user.getName(),
                                user.getEmail(), user.getStatus().name());
    }
}

// MapStruct — generates mapping code at compile time (zero reflection)
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponse toResponse(User user);

    @Mapping(target = "passwordHash", expression = "java(encode(req.password()))")
    User toEntity(CreateUserRequest req);
}
```

---

## 11.2 `LazyInitializationException`

This exception occurs when you access a lazy-loaded association outside of a transaction:

```java
// Scenario: transaction ends in the service layer, controller accesses lazy collection
@Service
@Transactional
public class UserService {
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }   // ← transaction ends here
}

@RestController
public class UserController {
    public UserResponse getUser(Long id) {
        User user = userService.findById(id);
        // transaction is closed!
        user.getOrders().size();  // ← LazyInitializationException: no session
    }
}
```

### Open Session In View (OSIV) — The Band-Aid

Spring Boot enables OSIV by default. It keeps the Hibernate session open for the entire request (through the controller and view rendering). This "fixes" the lazy loading exception but is a trap:

```yaml
spring:
  jpa:
    open-in-view: true  # Spring Boot default — CHANGE THIS TO FALSE
```

**Why OSIV is bad:**
1. Lazy loads triggered during serialization cause hidden N+1 queries
2. The DB connection is held for the entire HTTP request — poor connection pool utilization
3. Performance problems are hidden in production, hard to debug

```yaml
spring:
  jpa:
    open-in-view: false  # correct production setting
```

**Proper Fix:** Load everything you need within the transaction, map to a DTO:

```java
@Service
@Transactional(readOnly = true)  // readOnly = true for queries (performance hint)
public class UserService {
    public UserWithOrdersResponse findByIdWithOrders(Long id) {
        User user = userRepository.findByIdWithOrders(id);  // JOIN FETCH
        return userMapper.toResponseWithOrders(user);  // map to DTO inside transaction
    }   // ← DTO exits transaction, no lazy loading needed afterward
}
```

---

## 11.3 The Transaction Self-Invocation Problem

Covered in `04_data_layer.md` but worth restating — it's the most common Spring bug:

```java
@Service
public class InvoiceService {

    @Transactional
    public void generateInvoices() {
        List<Order> orders = orderRepository.findPending();
        orders.forEach(order -> processOrder(order));  // calls this.processOrder
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)  // BROKEN — never fires
    public void processOrder(Order order) {
        invoiceRepository.save(new Invoice(order));
        order.setStatus(OrderStatus.INVOICED);
        orderRepository.save(order);
    }
}
```

**Why:** `generateInvoices` calls `this.processOrder` — goes directly to the real object, not the proxy. The `@Transactional` on `processOrder` is an annotation on the real object's method, but without the proxy intercepting the call, the transaction behavior never fires.

**Fix — inject self:**
```java
@Service
public class InvoiceService {

    @Autowired @Lazy
    private InvoiceService self;  // proxy reference to this bean

    @Transactional
    public void generateInvoices() {
        orderRepository.findPending()
            .forEach(order -> self.processOrder(order));  // goes through proxy
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processOrder(Order order) { ... }
}
```

---

## 11.4 Connection Pool Exhaustion

Symptom: app hangs, requests time out, logs show `HikariPool-1 - Connection is not available, request timed out after 30000ms`.

Common causes:

```java
// Cause 1: Long transaction holding a connection
@Transactional
public void processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId);
    String result = externalService.call();  // 5 seconds — connection held entire time!
    order.update(result);
    orderRepository.save(order);
}

// Cause 2: Deadlock — two transactions waiting for each other
// Transaction A: locks User 1, waits for User 2
// Transaction B: locks User 2, waits for User 1
// Both hold connections indefinitely

// Cause 3: Pool size too small for traffic
// 50 concurrent requests, pool size 10 → 40 requests queue up
```

**Fix:**
1. Keep transactions short — no external HTTP calls inside `@Transactional`
2. Use optimistic locking to avoid deadlocks
3. Tune pool size based on measured concurrency

---

## 11.5 `@Transactional(readOnly = true)`

For read-only operations, this hint tells Hibernate to skip dirty checking and flush:

```java
@Service
public class UserService {

    @Transactional(readOnly = true)  // performance optimization for reads
    public List<UserResponse> findAll() {
        return userRepository.findAll()
            .stream()
            .map(userMapper::toResponse)
            .collect(Collectors.toList());
    }

    @Transactional  // default — read-write
    public UserResponse create(CreateUserRequest request) {
        User user = userRepository.save(userMapper.toEntity(request));
        return userMapper.toResponse(user);
    }
}
```

Benefits:
- Hibernate skips dirty checking (no memory overhead tracking changes)
- Some DBs route read-only transactions to read replicas

---

## 11.6 Pagination — Never Return All Records

```java
// DANGEROUS — loads entire table into memory
@GetMapping("/users")
public List<User> getAll() {
    return userRepository.findAll();  // 1,000,000 rows → OOM
}

// CORRECT — paginated
@GetMapping("/users")
public Page<UserResponse> getAll(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size) {
    return userRepository.findAll(PageRequest.of(page, size))
        .map(userMapper::toResponse);
}
```

---

## 11.7 Jackson Serialization Issues

### Circular Reference

```java
@Entity
public class Department {
    @OneToMany(mappedBy = "department")
    private List<Employee> employees;  // each employee has department reference
}

@Entity
public class Employee {
    @ManyToOne
    private Department department;  // circular: Department → Employee → Department → ...
}
```

Fix with `@JsonManagedReference` / `@JsonBackReference`:

```java
@OneToMany
@JsonManagedReference  // serialized normally
private List<Employee> employees;

@ManyToOne
@JsonBackReference    // not serialized (breaks the cycle)
private Department department;
```

Better fix: use DTOs — no circular reference possible since you control what's in the DTO.

### Unknown Properties

```java
// If client sends extra fields, Jackson throws by default
// Fix globally:
@Bean
public Jackson2ObjectMapperBuilderCustomizer jacksonCustomizer() {
    return builder -> builder.featuresToDisable(
        DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES
    );
}
```

### Date Serialization

```java
// Without config: dates serialize as arrays [2024, 1, 15]
// With config:
@Bean
public Jackson2ObjectMapperBuilderCustomizer dateFormatCustomizer() {
    return builder -> builder
        .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        .modules(new JavaTimeModule());  // Java 8+ date types
}
```

---

## 11.8 Startup Performance

Large Spring Boot apps with many beans can have slow startups. Common causes:

1. **Too many beans** — every `@Component` scanned adds to startup time
2. **Eager initialization of heavy clients** — DB, Redis, Kafka connections all open at startup
3. **`@PostConstruct` doing heavy work** — avoid complex logic in init methods

### Lazy Bean Initialization

```yaml
spring:
  main:
    lazy-initialization: true  # initialize beans only when first used
```

Good for dev/test — faster startup. Bad for production — first request is slow, and errors appear at request time not startup time.

Better: enable laziness selectively per bean:

```java
@Bean
@Lazy  // only initialized when first injected
public ExpensiveThirdPartyClient client() { ... }
```

---

## Tricky Interview Questions

**Q: Your app throws `LazyInitializationException` in production. How do you diagnose and fix it?**

1. First, check if OSIV is enabled (`spring.jpa.open-in-view`). If yes, that's masking other issues — disable it and fix the root cause.
2. Find the stack trace — it shows which lazy relationship was accessed.
3. Fix options:
   - Add `JOIN FETCH` in the repository query to load the association eagerly for this specific use case
   - Use `@EntityGraph` on the repository method
   - Load the lazy association inside a `@Transactional` service method before mapping to DTO
   - Map to DTO inside the transaction so no lazy loading escapes the transaction boundary

Never fix by making the relationship EAGER globally — that causes N+1 for all other queries.

---

**Q: How would you improve the performance of an endpoint that takes 3 seconds to respond?**

Systematic approach:
1. **Enable SQL logging** — is there an N+1 query? Are there missing indexes?
2. **Check what the 3 seconds is spent on** — DB query? External API? Computation?
3. **Database fixes**: add index on the filtered column, fix N+1 with JOIN FETCH, use projections instead of full entities
4. **Caching**: if data is read-frequently and changes rarely, add `@Cacheable` with Redis
5. **Async**: if calling an external service, can the call be made async? Can the result be pre-fetched?
6. **Pagination**: if loading many records, add pagination

---

**Q: What is the difference between `@Transactional(readOnly = true)` and not having `@Transactional` at all?**

Without `@Transactional`, each repository call opens and closes its own transaction. If your service method calls 3 repository methods, that's 3 separate transactions — no guarantees of consistency between them, and no first-level cache sharing.

With `@Transactional(readOnly = true)`, all repository calls in the service method run in one transaction — shared first-level cache (finding the same entity twice hits the cache, not the DB), consistent read snapshot, and Hibernate's dirty-checking skip optimization.

---

**Q: Your `@Scheduled` job runs every minute and takes 90 seconds to complete. What happens?**

With the default single-threaded scheduler and `fixedRate`, Spring waits for the running task to complete before scheduling the next. So the next run starts at minute 2.5 (1 min rate + 90s execution = 150s after start). Tasks don't overlap.

With `fixedDelay`, the next run starts 60s after the previous one ends — so every 150s.

If you use a custom `ThreadPoolTaskScheduler` with multiple threads, and the task takes longer than the interval, multiple instances of the task will run concurrently — which may cause data consistency issues.

**Fix for long-running jobs:** Use `fixedDelay` (not `fixedRate`) to ensure the previous run finishes before the next starts, or use ShedLock to ensure only one instance runs at a time in a cluster.
