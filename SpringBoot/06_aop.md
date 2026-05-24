# 06 — AOP (Aspect-Oriented Programming)

---

## 6.1 The Problem AOP Solves

Imagine you need to log the execution time of every service method:

```java
// Without AOP — duplicated boilerplate everywhere
@Service
public class UserService {
    public User findById(Long id) {
        long start = System.currentTimeMillis();
        log.info("Entering findById with id={}", id);
        try {
            User result = userRepository.findById(id).orElseThrow();
            return result;
        } finally {
            log.info("findById took {}ms", System.currentTimeMillis() - start);
        }
    }
    // Same boilerplate in 50 other methods...
}
```

AOP extracts this **cross-cutting concern** (logging, security, transactions, retry) into a single class called an **Aspect**. All methods get the behavior automatically without touching them.

---

## 6.2 Core Concepts

| Term | Meaning | Example |
|---|---|---|
| **Aspect** | Class containing the cross-cutting logic | `LoggingAspect` |
| **Advice** | The code that runs (the "what") | `@Around`, `@Before`, `@After` |
| **Pointcut** | Expression selecting which methods to intercept (the "where") | `execution(* com.example.service.*.*(..))` |
| **Join Point** | A specific method execution being intercepted | `UserService.findById(1L)` |
| **Target** | The original bean being advised | `UserService` |
| **Proxy** | The wrapper Spring creates around the target | CGLIB/JDK proxy wrapping `UserService` |

---

## 6.3 Advice Types

```
Method starts
    ↓
@Before ← runs before method
    ↓
Method body executes
    ↓
@AfterReturning ← runs only on successful return
@AfterThrowing  ← runs only if exception thrown
@After          ← runs always (like finally)
    ↓
@Around ← wraps everything above (most powerful)
```

### `@Before`

```java
@Aspect
@Component
public class SecurityAspect {

    @Before("execution(* com.example.service.AdminService.*(..))")
    public void checkAdminAccess(JoinPoint joinPoint) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (!auth.getAuthorities().contains(new SimpleGrantedAuthority("ROLE_ADMIN"))) {
            throw new AccessDeniedException("Admin access required for: "
                + joinPoint.getSignature().getName());
        }
    }
}
```

### `@AfterReturning`

```java
@Aspect
@Component
public class AuditAspect {

    @AfterReturning(
        pointcut = "execution(* com.example.service.UserService.create(..))",
        returning = "result"  // binds the return value
    )
    public void auditCreate(JoinPoint joinPoint, Object result) {
        auditLog.record("User created", result);
    }
}
```

### `@AfterThrowing`

```java
@Aspect
@Component
public class ErrorAspect {

    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "ex"  // binds the exception
    )
    public void logException(JoinPoint joinPoint, Exception ex) {
        log.error("Exception in {}.{}: {}",
            joinPoint.getTarget().getClass().getSimpleName(),
            joinPoint.getSignature().getName(),
            ex.getMessage());
        alertService.notify(ex);  // send alert to PagerDuty
    }
}
```

### `@Around` — Most Powerful

```java
@Aspect
@Component
public class TimingAspect {

    // @Around must use ProceedingJoinPoint (not JoinPoint)
    @Around("execution(* com.example.service.*.*(..))")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        String methodName = pjp.getSignature().toShortString();
        long start = System.currentTimeMillis();

        try {
            Object result = pjp.proceed();  // call the actual method
            log.info("{} completed in {}ms", methodName, System.currentTimeMillis() - start);
            return result;
        } catch (Throwable ex) {
            log.error("{} threw {} after {}ms",
                methodName, ex.getClass().getSimpleName(),
                System.currentTimeMillis() - start);
            throw ex;  // re-throw, don't swallow
        }
    }
}
```

---

## 6.4 Pointcut Expressions

The pointcut expression language selects which methods to intercept:

### `execution` — Most Common

```java
// execution(modifiers? return-type declaring-type?.method-name(params) throws?)
// * = any single element
// .. = any number of elements

// Any public method in any class
"execution(public * *(..))"

// Any method in UserService
"execution(* com.example.service.UserService.*(..))"

// Any method in any class in the service package (not subpackages)
"execution(* com.example.service.*.*(..))"

// Any method in any class in the service package AND subpackages
"execution(* com.example.service..*.*(..))"

// Methods named "find*" with any parameters
"execution(* find*(..))"

// Methods taking exactly one String parameter
"execution(* *(String))"

// Methods taking a Long as first parameter and anything else
"execution(* *(Long, ..))"
```

### `@annotation` — Match by Annotation

```java
// Any method annotated with @Retryable
"@annotation(com.example.annotation.Retryable)"

// Usage:
@Aspect
@Component
public class RetryAspect {
    @Around("@annotation(retryable)")  // binds the annotation instance
    public Object retry(ProceedingJoinPoint pjp, Retryable retryable) throws Throwable {
        int maxAttempts = retryable.maxAttempts();
        // ...
    }
}
```

### `@within` — Match by Class Annotation

```java
// Any method in a class annotated with @Service
"@within(org.springframework.stereotype.Service)"
```

### Named Pointcuts (Reusable)

```java
@Aspect
@Component
public class AppAspects {

    // Define reusable pointcuts
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    @Pointcut("execution(* com.example.repository.*.*(..))")
    public void repositoryLayer() {}

    @Pointcut("serviceLayer() || repositoryLayer()")
    public void serviceOrRepository() {}

    // Use named pointcuts in advice
    @Around("serviceLayer()")
    public Object timeServiceMethods(ProceedingJoinPoint pjp) throws Throwable {
        // ...
    }

    @Before("serviceOrRepository()")
    public void logEntry(JoinPoint jp) {
        // ...
    }
}
```

---

## 6.5 How AOP Works — The Proxy Mechanism

Spring AOP is **proxy-based**, not bytecode-level (unlike AspectJ). Spring wraps your bean in a proxy at startup.

### JDK Dynamic Proxy vs CGLIB

```
JDK Dynamic Proxy:
  - Used when the bean implements an interface
  - Proxy is a new class implementing the same interface
  - Fast, but only intercepts interface methods

CGLIB Proxy:
  - Used when the bean does NOT implement an interface (or Spring Boot default)
  - Proxy is a subclass of your bean
  - Can intercept any non-final, non-private method
  - Spring Boot uses CGLIB by default even for interface-backed beans
```

```java
// What Spring actually creates at runtime:
@Service
public class UserService {  // your class
    public User findById(Long id) { ... }
}

// Spring auto-generates a proxy class like:
class UserService$$SpringCGLIB extends UserService {
    @Override
    public User findById(Long id) {
        // pre-processing (before advice)
        User result = super.findById(id);  // call your actual method
        // post-processing (after advice)
        return result;
    }
}

// What gets injected into other beans:
@Service
public class OrderService {
    @Autowired
    private UserService userService;  // actually gets UserService$$SpringCGLIB
}
```

### Why Self-Invocation Breaks AOP

```java
@Service
public class OrderService {
    public void placeOrder(Order order) {
        this.validateOrder(order);  // "this" refers to real object, not the proxy
    }

    @Transactional  // this advice NEVER fires when called via this.validateOrder()
    public void validateOrder(Order order) { ... }
}

// External caller:
orderService.placeOrder(order);
// → hits proxy → calls placeOrder on real object
// → inside placeOrder, this.validateOrder bypasses proxy
// → @Transactional on validateOrder has no effect
```

**Fix:**
```java
// Option 1: Inject the service into itself (self-injection)
@Service
public class OrderService {
    @Autowired
    @Lazy  // prevents circular dependency
    private OrderService self;

    public void placeOrder(Order order) {
        self.validateOrder(order);  // goes through the proxy
    }
}

// Option 2: Extract to a separate class
@Service
public class OrderValidationService {
    @Transactional
    public void validateOrder(Order order) { ... }
}
```

---

## 6.6 Practical AOP Examples

### Retry Logic

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Retryable {
    int maxAttempts() default 3;
    long delayMs() default 1000;
}

@Aspect
@Component
public class RetryAspect {

    @Around("@annotation(retryable)")
    public Object retry(ProceedingJoinPoint pjp, Retryable retryable) throws Throwable {
        int attempts = 0;
        Throwable lastException = null;

        while (attempts < retryable.maxAttempts()) {
            try {
                return pjp.proceed();
            } catch (RuntimeException ex) {
                lastException = ex;
                attempts++;
                if (attempts < retryable.maxAttempts()) {
                    Thread.sleep(retryable.delayMs());
                }
            }
        }
        throw lastException;
    }
}

// Usage:
@Retryable(maxAttempts = 3, delayMs = 500)
public String callExternalApi(String endpoint) { ... }
```

### Method-Level Authorization

```java
@Aspect
@Component
public class AuthorizationAspect {

    @Before("@annotation(requiredRole)")
    public void checkRole(JoinPoint jp, RequiredRole requiredRole) {
        String role = requiredRole.value();
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        boolean hasRole = auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + role));
        if (!hasRole) {
            throw new AccessDeniedException("Missing role: " + role);
        }
    }
}

// Usage:
@RequiredRole("ADMIN")
public void deleteUser(Long id) { ... }
```

---

## Tricky Interview Questions

**Q: How does `@Transactional` work internally? Is it AOP?**

Yes. `@Transactional` is implemented using Spring AOP. The `TransactionInterceptor` is a `MethodInterceptor` (Around advice) that:
1. Begins a transaction before the method
2. Commits the transaction after successful return
3. Rolls back on unchecked exception

Spring registers a `BeanPostProcessor` that wraps any bean with `@Transactional` methods in a proxy. The proxy calls `TransactionInterceptor` before delegating to the real bean.

---

**Q: What happens if you put `@Around` on a method but forget to call `pjp.proceed()`?**

The actual method **never executes**. The around advice completely controls the execution flow. If you don't call `proceed()`, the target method is skipped and the return value of the advice method is returned instead.

This is actually useful for mocking, short-circuit evaluation, or feature flags:

```java
@Around("execution(* com.example.feature.experimental.*.*(..))")
public Object checkFeatureFlag(ProceedingJoinPoint pjp) throws Throwable {
    if (!featureFlags.isEnabled("experimental")) {
        return null;  // skip the method entirely
    }
    return pjp.proceed();
}
```

---

**Q: Can Spring AOP intercept private methods?**

No. Spring AOP is proxy-based, and proxies (whether JDK or CGLIB) can only override **public and protected** methods. Private methods are not inherited by subclasses or overridable via interfaces — the proxy cannot intercept them.

Use AspectJ weaving (compile-time or load-time) if you need to intercept private methods.

---

**Q: Two `@Around` aspects are defined on the same method. Which runs first?**

The order is undefined unless you use `@Order`. Lower order values run first (outermost):

```java
@Aspect
@Order(1)  // runs outermost — first Before, last After
@Component
public class LoggingAspect { ... }

@Aspect
@Order(2)  // runs inner — second Before, first After
@Component
public class TransactionAspect { ... }

// Execution order:
// LoggingAspect.@Before → TransactionAspect.@Before → method → TransactionAspect.@After → LoggingAspect.@After
```

---

**Q: What is the difference between Spring AOP and AspectJ?**

| | Spring AOP | AspectJ |
|---|---|---|
| Mechanism | Runtime proxy (CGLIB/JDK) | Bytecode weaving (compile/load-time) |
| Scope | Spring-managed beans only | Any Java class, including non-Spring |
| Methods | Public/protected only | Any method including private |
| Performance | Slight proxy overhead | Near-zero overhead after weaving |
| Setup | Zero config, works out of the box | Requires AspectJ compiler or agent |
| Use for | Most real-world Spring use cases | Complex cross-cutting in large systems |

Spring Boot uses Spring AOP by default. AspectJ is added explicitly when you need its full power.
