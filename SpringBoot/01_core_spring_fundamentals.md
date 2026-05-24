# 01 — Core Spring Fundamentals

---

## 1.1 IoC (Inversion of Control) & Dependency Injection

### What is IoC?

In traditional code, your class creates its own dependencies:

```java
// Without IoC — tightly coupled
public class OrderService {
    private PaymentService paymentService = new PaymentService(); // hardwired
}
```

With IoC, the container creates and injects dependencies. Your class just declares what it needs:

```java
// With IoC — loosely coupled
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService; // Spring injects this
    }
}
```

**Why it matters:** You can swap `PaymentService` with a mock in tests, or swap the real implementation without touching `OrderService`. The class has no idea who creates its dependency — that control is inverted to the container.

---

### Three Types of Dependency Injection

#### 1. Constructor Injection (Preferred)

```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final NotificationService notificationService;

    // @Autowired is optional here if there's only one constructor (Spring 4.3+)
    public OrderService(PaymentService paymentService, NotificationService notificationService) {
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
}
```

**Why preferred:**
- Dependencies are `final` — object is fully initialized and immutable
- Fails fast at startup if a dependency is missing
- Makes circular dependencies visible at compile time
- No reflection needed — easy to test with `new OrderService(mockPayment, mockNotification)`

#### 2. Setter Injection

```java
@Service
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Use only for optional dependencies. The object can exist in a partially initialized state.

#### 3. Field Injection (Avoid in Production Code)

```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService; // Spring uses reflection to set this
}
```

**Problems:** Can't make `final`, can't test without Spring context or reflection hacks, hides dependencies (constructor doesn't show them), breaks if Spring container isn't present.

---

## 1.2 The Spring Container

### BeanFactory vs ApplicationContext

| | BeanFactory | ApplicationContext |
|---|---|---|
| Bean creation | Lazy (on first `getBean()`) | Eager (at startup by default) |
| AOP support | No | Yes |
| Event publishing | No | Yes (`ApplicationEventPublisher`) |
| Internationalization | No | Yes (`MessageSource`) |
| Usage | Rarely used directly | Always use this |

```java
// How ApplicationContext is created in a non-Boot Spring app
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
OrderService service = ctx.getBean(OrderService.class);
```

In Spring Boot, the context is created automatically when you call `SpringApplication.run()`.

### What Happens at Startup

1. Component scan — finds all `@Component`, `@Service`, `@Repository`, etc.
2. Reads `@Configuration` classes for `@Bean` definitions
3. Resolves dependencies between beans (builds a dependency graph)
4. Instantiates beans (in dependency order)
5. Runs `BeanPostProcessor` hooks (AOP proxies are created here)
6. Calls `@PostConstruct` methods
7. Publishes `ApplicationReadyEvent`

---

## 1.3 Bean Lifecycle

### Full Lifecycle Sequence

```
1. BeanDefinition registered (component scan / @Bean)
2. Bean instantiated (constructor called)
3. Dependencies injected (properties populated)
4. BeanNameAware.setBeanName() called (if implemented)
5. BeanFactoryAware.setBeanFactory() called (if implemented)
6. ApplicationContextAware.setApplicationContext() called (if implemented)
7. BeanPostProcessor.postProcessBeforeInitialization() — AOP proxy creation happens here
8. @PostConstruct method called
9. InitializingBean.afterPropertiesSet() called (if implemented)
10. @Bean(initMethod = "init") called (if specified)
11. BeanPostProcessor.postProcessAfterInitialization()
12. Bean is ready to use
--- bean lives here ---
13. ApplicationContext.close() triggered
14. @PreDestroy method called
15. DisposableBean.destroy() called (if implemented)
16. @Bean(destroyMethod = "cleanup") called (if specified)
```

### Practical Example

```java
@Component
public class DatabaseConnectionPool {
    private Connection connection;

    @PostConstruct
    public void init() {
        // Called after all dependencies are injected
        // Safe to use injected fields here
        this.connection = openConnection();
        System.out.println("Connection pool initialized");
    }

    @PreDestroy
    public void cleanup() {
        // Called before the bean is removed from context
        if (connection != null) connection.close();
        System.out.println("Connection pool closed");
    }
}
```

### BeanPostProcessor — How AOP and Transactions Work

```java
// Spring internally does something like this for @Transactional:
public class TransactionProxyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (hasTransactionalAnnotation(bean)) {
            return createTransactionalProxy(bean); // wraps bean in a proxy
        }
        return bean; // return original bean unchanged
    }
}
```

This is why `@Transactional` works without you writing any proxy code — Spring replaces your bean with a proxy during context initialization.

---

## 1.4 AOP (Aspect-Oriented Programming)

### The Problem AOP Solves

In a production codebase, many methods need the same boilerplate — logging, timing, transaction management, security checks. Without AOP you'd copy-paste that code everywhere. With AOP you write it **once** and Spring automatically weaves it into the right places.

These are called **cross-cutting concerns** — they don't belong to any single class but are needed across many:
- Logging every service method call
- Starting / committing a database transaction
- Checking if the user has permission
- Measuring method execution time

### How Spring Does It — Proxies

When the `ApplicationContext` starts up (step 5 from section 1.2), instead of giving you the real `OrderService` bean, Spring gives you a **proxy** — a transparent wrapper that intercepts method calls:

```
Your code calls orderService.placeOrder()
        ↓
AOP Proxy intercepts the call
        ↓
Proxy runs "before" logic: start transaction, log the call
        ↓
Proxy delegates to the real OrderService.placeOrder()
        ↓
Proxy runs "after" logic: commit transaction, log result
        ↓
Result returned to your code
```

You never see the proxy — it behaves exactly like the original object.

### Key Terminology

| Term | Meaning | Plain English |
|---|---|---|
| **Aspect** | Class containing the cross-cutting logic | Your "logging module" |
| **Advice** | The actual code to run | The log statement itself |
| **Pointcut** | Rule for *where* to apply advice | "Every `@Service` method" |
| **Join Point** | A specific method call being intercepted | `orderService.placeOrder()` at runtime |

### Real Example — `@Transactional`

The most common use of AOP in Spring is `@Transactional`:

```java
@Service
public class OrderService {

    @Transactional  // ← AOP proxy handles all transaction logic
    public void placeOrder(Order order) {
        // proxy starts a DB transaction BEFORE this runs
        inventoryRepo.decrementStock(order.getItemId());
        paymentRepo.charge(order.getUserId(), order.getAmount());
        // proxy commits on success, rolls back on RuntimeException
    }
}
```

You wrote zero transaction management code. Spring's AOP proxy handled it entirely via `TransactionProxyBeanPostProcessor` (shown in section 1.3).

### Writing a Custom Aspect

```java
@Aspect
@Component
public class ExecutionTimeAspect {

    // Pointcut: match every public method in any class under com.example.service
    @Around("execution(* com.example.service.*.*(..))")
    public Object measureTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();

        Object result = joinPoint.proceed(); // call the real method

        long elapsed = System.currentTimeMillis() - start;
        System.out.printf("%s took %dms%n", joinPoint.getSignature(), elapsed);
        return result;
    }
}
```

`@Around` is the most powerful advice type — it wraps the method entirely. Others:
- `@Before` — runs before the method
- `@After` — runs after (regardless of outcome)
- `@AfterReturning` — runs only on success
- `@AfterThrowing` — runs only on exception

### Critical Gotcha — Self-Invocation Bypasses the Proxy

AOP proxies only intercept calls that come **through Spring** (i.e., through the injected bean reference). If you call a method on `this` inside the same class, the proxy is bypassed:

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) { ... }

    public void batchOrders(List<Order> orders) {
        for (Order o : orders) {
            this.placeOrder(o); // ← calls real method directly, proxy is SKIPPED
            // no transaction started — this is a very common production bug
        }
    }
}
```

**Fix:** Extract `placeOrder` into a separate bean and inject it, or restructure the logic so the transactional method is called from outside the class.

---

## 1.5 Bean Scopes

### Singleton (Default)

```java
@Component
// @Scope("singleton") — this is the default, you don't need to write it
public class UserService {
    // One instance shared across the entire application
}
```

Singletons must be **stateless** — if you store state in a field, all threads share it.

### Prototype

```java
@Component
@Scope("prototype")
public class ReportGenerator {
    private List<String> lines = new ArrayList<>(); // each caller gets their own instance

    public void addLine(String line) { lines.add(line); }
}
```

Spring creates a new instance every time a prototype bean is requested. Spring does **not** manage the lifecycle of prototype beans — `@PreDestroy` is NOT called.

### Web Scopes

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String correlationId = UUID.randomUUID().toString();
}
```

- `request` — one instance per HTTP request
- `session` — one instance per HTTP session
- `application` — one instance per `ServletContext`

`proxyMode = ScopedProxyMode.TARGET_CLASS` is required when injecting a short-lived scope into a longer-lived one (e.g., request-scoped into a singleton).

### The Prototype-in-Singleton Trap

```java
@Component  // singleton
public class OrderProcessor {
    @Autowired
    private ReportGenerator reportGenerator; // prototype, but injected once at startup!

    public void process() {
        // reportGenerator is the SAME instance every time — it's not prototype here
        reportGenerator.addLine("Processing order");
    }
}
```

**Fix using `ObjectProvider`:**

```java
@Component
public class OrderProcessor {
    @Autowired
    private ObjectProvider<ReportGenerator> reportGeneratorProvider;

    public void process() {
        ReportGenerator generator = reportGeneratorProvider.getObject(); // fresh instance each time
        generator.addLine("Processing order");
    }
}
```

---

## 1.6 Stereotype Annotations

```
@Component          — generic Spring-managed component
├── @Service        — business logic layer (no extra behavior, just semantic clarity)
├── @Repository     — data access layer (adds exception translation)
└── @Controller     — web layer (handles HTTP, works with view resolvers)
    └── @RestController — @Controller + @ResponseBody (returns JSON/XML directly)
```

### `@Repository` Exception Translation

```java
@Repository
public class UserRepository {
    // If this throws a JDBC SQLException, Spring translates it to
    // a DataAccessException subclass — consistent exception hierarchy
    // regardless of which DB driver you use
    public User findById(Long id) { ... }
}
```

Without `@Repository`, a JDBC exception leaks the underlying driver details (MySQL exception, Oracle exception, etc.). With `@Repository`, you always catch `DataAccessException`.

---

## Tricky Interview Questions

**Q: What happens if two beans of the same type exist and you try to `@Autowired` one?**

Spring throws `NoUniqueBeanDefinitionException`. Fix: add `@Primary` to the preferred bean, or use `@Qualifier("beanName")` at the injection point.

```java
@Bean @Primary
public PaymentService stripePayment() { return new StripePaymentService(); }

@Bean
public PaymentService paypalPayment() { return new PayPalPaymentService(); }

// Injection site — no @Qualifier needed because of @Primary
@Autowired
private PaymentService paymentService; // gets StripePaymentService
```

---

**Q: What's the difference between `@Component` on a class and `@Bean` in a `@Configuration` class?**

- `@Component` — Spring discovers the class via component scan; Spring controls instantiation
- `@Bean` — you write the factory method; you control how the object is created (useful for third-party classes you can't annotate)

```java
// Can't put @Component on a class from an external library
@Configuration
public class AppConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    }
}
```

---

**Q: Can a `@Configuration` class itself be a Spring bean?**

Yes. `@Configuration` classes are registered as beans. Spring creates a CGLIB proxy for `@Configuration` classes to ensure that `@Bean` methods return the same singleton instance when called multiple times:

```java
@Configuration
public class AppConfig {
    @Bean
    public ServiceA serviceA() {
        return new ServiceA(serviceB()); // calls serviceB()
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB();
    }
}
// serviceB() is called twice but returns the SAME instance — CGLIB proxy intercepts it
```

If you use `@Configuration(proxyBeanMethods = false)`, the CGLIB proxy is skipped — each call to `serviceB()` creates a new instance.

---

**Q: What is circular dependency and how does Spring handle it?**

When Bean A depends on Bean B and Bean B depends on Bean A.

- **With constructor injection:** Spring detects this at startup and throws `BeanCurrentlyInCreationException` — this is the correct behavior, it forces you to fix the design
- **With field/setter injection:** Spring resolves it by injecting a partially initialized bean (risky)

The real fix is to redesign: extract a third bean that both A and B depend on, or reconsider the responsibility boundaries.

---

**Q: `@PostConstruct` vs constructor — what's the difference for initialization logic?**

In the constructor, dependencies haven't been injected yet (for field injection). `@PostConstruct` fires after all dependencies are wired, so it's safe to use them:

```java
@Component
public class CacheWarmer {
    @Autowired
    private UserRepository userRepository; // null in constructor with field injection

    @PostConstruct
    public void warmUp() {
        // userRepository is available here
        userRepository.findAll().forEach(cache::put);
    }
}
```

With constructor injection, this is a non-issue — you can safely use injected fields in the constructor body itself.
