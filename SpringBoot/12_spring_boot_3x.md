# 12 — Spring Boot 3.x / Modern Changes

---

## 12.1 Why Spring Boot 3 Matters

Spring Boot 3 (released November 2022) is a major version with breaking changes. Most companies running Java 17+ are migrating to it. Knowing what changed shows you're current.

---

## 12.2 Java 17 Minimum

Spring Boot 3 requires **Java 17 at minimum**. Boot 2.x supported Java 8+.

Key Java 17 features now usable in Spring Boot 3 apps:

### Records

```java
// Instead of a DTO class with constructor, getters, equals, hashCode:
public class UserResponse {
    private final Long id;
    private final String name;
    public UserResponse(Long id, String name) { ... }
    // + equals, hashCode, toString...
}

// Use a record — all of the above is auto-generated:
public record UserResponse(Long id, String name) {}

// Works perfectly with Jackson, @RequestBody, @ResponseBody, Spring Data projections
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) {
    return new UserResponse(user.getId(), user.getName());
}
```

### Sealed Classes (Domain modeling)

```java
// Sealed hierarchy for payment methods
public sealed class PaymentMethod permits CreditCard, BankTransfer, Crypto {
    public abstract BigDecimal getFee(BigDecimal amount);
}

public final class CreditCard extends PaymentMethod {
    public BigDecimal getFee(BigDecimal amount) { return amount.multiply(new BigDecimal("0.03")); }
}

public final class BankTransfer extends PaymentMethod {
    public BigDecimal getFee(BigDecimal amount) { return BigDecimal.ZERO; }
}

// Pattern matching in switch (Java 21):
BigDecimal fee = switch (method) {
    case CreditCard cc -> cc.getFee(amount);
    case BankTransfer bt -> bt.getFee(amount);
    case Crypto c -> c.getFee(amount);
};
```

### Text Blocks

```java
// Multi-line strings without escaping
String query = """
        SELECT u.id, u.name, o.total
        FROM users u
        JOIN orders o ON u.id = o.user_id
        WHERE u.status = 'ACTIVE'
        ORDER BY o.created_at DESC
        """;
```

---

## 12.3 Jakarta EE 9+ — Package Rename

The most disruptive change: all `javax.*` packages renamed to `jakarta.*`. This affects every file in your project that uses JPA, Servlets, Bean Validation, or JAX-RS.

```java
// Spring Boot 2.x — javax namespace
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.validation.constraints.NotBlank;
import javax.servlet.http.HttpServletRequest;
import javax.transaction.Transactional;

// Spring Boot 3.x — jakarta namespace
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.validation.constraints.NotBlank;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.transaction.Transactional;
```

**Migration tip:** This is a find-and-replace across the project. IntelliJ has a built-in migration tool: `Refactor → Migrate Packages and Classes → Java EE to Jakarta EE`.

---

## 12.4 Auto-Configuration Discovery Changes

In Boot 2.x, auto-configuration classes were listed in `META-INF/spring.factories`:

```properties
# Boot 2.x
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration,\
  com.example.AnotherAutoConfiguration
```

In Boot 3.x, this moved to `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```
# Boot 3.x
com.example.MyAutoConfiguration
com.example.AnotherAutoConfiguration
```

If you write custom auto-configuration libraries (or use third-party starters), they must update to the new location to work with Boot 3.

---

## 12.5 Problem Details for HTTP APIs (RFC 7807)

Spring Boot 3 has native support for the HTTP Problem Details standard (`application/problem+json`). Instead of a custom error response format, you get a standardized one:

```json
{
  "type": "https://example.com/problems/user-not-found",
  "title": "User Not Found",
  "status": 404,
  "detail": "User with id 42 does not exist",
  "instance": "/api/users/42"
}
```

Enable it:

```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

Spring automatically converts `ResponseEntityExceptionHandler`'s exceptions to problem details format. For custom exceptions:

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ProblemDetail handleUserNotFound(UserNotFoundException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("User Not Found");
        problem.setType(URI.create("https://api.example.com/problems/user-not-found"));
        problem.setProperty("userId", ex.getUserId());  // custom extension fields
        return problem;
    }
}
```

---

## 12.6 Virtual Threads (Java 21 + Spring Boot 3.2)

Traditional Spring MVC uses one OS thread per HTTP request. Under high load, threads block waiting for I/O (DB queries, HTTP calls) but the OS thread is held.

Virtual threads (Project Loom, Java 21) are lightweight threads managed by the JVM, not the OS. You can have millions of them. A virtual thread that blocks on I/O automatically parks (releases the carrier thread) and resumes when I/O is ready.

```yaml
# Spring Boot 3.2+ — one config line enables virtual threads for everything
spring:
  threads:
    virtual:
      enabled: true
```

This makes Tomcat use virtual threads for request handling and `@Async` methods. Your existing blocking code gets throughput benefits without rewriting to reactive:

```java
// This blocking code now works at scale with virtual threads:
@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepository.findById(id).orElseThrow();   // blocks, but virtual thread parks
    String enriched = externalApi.fetch(id);                  // blocks again, but parks
    return userMapper.toResponse(user, enriched);
}
```

**Who needs WebFlux?** Virtual threads reduce the advantage of the reactive model for I/O-bound workloads. WebFlux still has an edge for CPU-bound streaming or backpressure scenarios.

---

## 12.7 Spring Boot 3.2+ Native Image (GraalVM)

Compile your Spring Boot app to a native binary — no JVM startup time, very low memory:

```
Traditional JVM: startup 3-5 seconds, uses 256MB RAM
GraalVM Native: startup <100ms, uses 50MB RAM
```

```bash
# Maven — build native image
./mvnw -Pnative native:compile

# Run the native binary
./target/my-app
```

**Restrictions:**
- Reflection must be declared ahead of time (Spring handles most of it automatically)
- Dynamic class loading is limited
- Slower build times (minutes, not seconds)
- Some libraries aren't compatible

Most Spring Boot features work with native image out of the box. Best for: serverless functions, containers with rapid scale-out.

---

## 12.8 Observability Improvements

Spring Boot 3 replaced Spring Cloud Sleuth with **Micrometer Tracing**:

```xml
<!-- Boot 3 tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
    <!-- or micrometer-tracing-bridge-otel for OpenTelemetry -->
</dependency>
```

The `@Observed` annotation (Boot 3.2+) automatically instruments any Spring bean method:

```java
@Service
@Observed(name = "user-service", contextualName = "user-service")
public class UserService {
    // All methods automatically traced + metrics recorded
}

// Or per method:
@Observed(name = "create-user", lowCardinalityKeyValues = {"layer", "service"})
public UserResponse createUser(CreateUserRequest request) { ... }
```

---

## 12.9 Summary Table: Boot 2.x vs Boot 3.x

| Feature | Boot 2.x | Boot 3.x |
|---|---|---|
| Min Java | Java 8 | **Java 17** |
| Namespace | `javax.*` | **`jakarta.*`** |
| Auto-config location | `spring.factories` | **`AutoConfiguration.imports`** |
| Actuator | Micrometer + Sleuth | **Micrometer Tracing** |
| Tracing | Spring Cloud Sleuth | **Micrometer Tracing (built-in)** |
| HTTP error format | Custom | **Problem Details (RFC 7807)** |
| Virtual threads | No | **Yes (Boot 3.2 + Java 21)** |
| Native image | Limited | **First-class support** |
| Spring Security | `WebSecurityConfigurerAdapter` | **`SecurityFilterChain` bean** |

---

## Tricky Interview Questions

**Q: You're migrating from Spring Boot 2.7 to Boot 3. What are the first three things you check?**

1. **Java version** — upgrade to Java 17+ if on Java 11 or 8
2. **Package imports** — replace all `javax.*` with `jakarta.*` (use IDE migration tool)
3. **Third-party dependency compatibility** — check that all your starters/libraries have Boot 3 compatible versions (many libraries had to release Boot 3 versions separately)

Additional: check for removed/renamed `WebSecurityConfigurerAdapter` usage and update to `SecurityFilterChain` bean configuration.

---

**Q: What is the difference between virtual threads and reactive programming (WebFlux)?**

Both solve the "thread-per-request bottleneck":
- **Reactive (WebFlux):** Non-blocking I/O, event loop model. You rewrite your code in a reactive style (`Mono`, `Flux`). Very efficient but complex to write and debug.
- **Virtual threads:** JVM-managed lightweight threads. Your code stays blocking/synchronous but doesn't waste OS threads. Simpler — no paradigm shift required.

For most I/O-bound apps, virtual threads on Spring MVC is the easier path to the same throughput improvement. WebFlux has an advantage for streaming, backpressure, and very high connection counts (100k+).

---

**Q: `@Observed` vs `@Timed` — what's the difference in Boot 3?**

- `@Timed` (Micrometer): records execution time as a `Timer` metric
- `@Observed` (Spring Boot 3.2+): records both a **Timer metric AND a distributed trace span** in one annotation

`@Observed` is the preferred choice in Boot 3 because it instruments for both observability pillars (metrics + tracing) simultaneously, using the same annotation.
