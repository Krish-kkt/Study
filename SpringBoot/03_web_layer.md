# 03 — Web Layer (REST APIs)

---

## 3.1 How a Request Flows Through Spring MVC

Before writing controllers, understand the journey every HTTP request takes:

```
Client
  → Filter Chain (Auth filter, CORS filter, logging filter...)
    → DispatcherServlet (front controller — routes all requests)
      → HandlerMapping (finds which controller method handles this URL)
        → HandlerAdapter (calls the controller method)
          → Controller Method (your code)
            → HttpMessageConverter (serializes return value to JSON)
      → ExceptionResolver (if exception thrown)
  → Response
```

**DispatcherServlet** is the entry point. There is exactly one per application. Everything else is plugged into it.

---

## 3.2 Controller Basics

```java
@RestController                    // @Controller + @ResponseBody
@RequestMapping("/api/v1/users")   // base path for all methods
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public UserResponse getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }

    @PutMapping("/{id}")
    public UserResponse updateUser(@PathVariable Long id,
                                   @Valid @RequestBody UpdateUserRequest request) {
        return userService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### `@Controller` vs `@RestController`

```java
@Controller  // Returns view names (templates like Thymeleaf)
public class ViewController {
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("user", "Alice");
        return "home"; // resolves to templates/home.html
    }
}

@RestController  // Returns data directly as JSON/XML
public class ApiController {
    @GetMapping("/users")
    public List<User> users() {
        return List.of(new User("Alice")); // serialized to JSON automatically
    }
}
```

---

## 3.3 Request Binding

### `@PathVariable`

Extracts values from the URL path:

```java
// URL: GET /orders/42/items/7
@GetMapping("/orders/{orderId}/items/{itemId}")
public OrderItem getItem(@PathVariable Long orderId, @PathVariable Long itemId) {
    return orderService.findItem(orderId, itemId);
}
```

### `@RequestParam`

Extracts from query string:

```java
// URL: GET /users?page=0&size=20&sort=name&active=true
@GetMapping("/users")
public Page<UserResponse> listUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size,
    @RequestParam(required = false) String sort,
    @RequestParam(defaultValue = "true") boolean active) {
    return userService.list(page, size, sort, active);
}
```

### `@RequestBody`

Deserializes JSON body into a Java object via Jackson:

```java
// POST /users with body: {"name": "Alice", "email": "alice@example.com"}
@PostMapping("/users")
public ResponseEntity<UserResponse> createUser(@Valid @RequestBody CreateUserRequest request) {
    UserResponse created = userService.create(request);
    URI location = URI.create("/users/" + created.getId());
    return ResponseEntity.created(location).body(created);
}
```

Jackson uses field names or `@JsonProperty`. It ignores unknown fields by default if configured.

### `@RequestHeader`

```java
@GetMapping("/data")
public DataResponse getData(
    @RequestHeader("X-Correlation-ID") String correlationId,
    @RequestHeader(value = "X-API-Version", defaultValue = "1") String version) {
    log.info("Request {} on API v{}", correlationId, version);
    return dataService.fetch();
}
```

---

## 3.4 `ResponseEntity`

### What is it and why does it exist?

When a controller method returns a plain object like `UserResponse`, Spring automatically wraps it in a `200 OK` response. That's fine for simple reads. But HTTP has a rich vocabulary — `201 Created`, `204 No Content`, `404 Not Found`, `Location` headers — and raw return types can't express any of that.

`ResponseEntity<T>` is Spring's way of letting you control the **full HTTP response**: status code, headers, and body, all in one place. Think of it as a wrapper that says "I want to decide exactly what goes back to the client."

```
ResponseEntity<T>
├── Status Code  (200, 201, 204, 404, ...)
├── Headers      (Location, Cache-Control, X-Custom-Header, ...)
└── Body         (your object T, or Void if no body)
```

---

### The builder API — how to construct responses

Spring provides a fluent builder. Here are the patterns you'll use constantly:

```java
// 200 OK with body
ResponseEntity.ok(userResponse);

// 200 OK with body (explicit)
ResponseEntity.ok().body(userResponse);

// 201 Created — must include a Location header (REST best practice)
ResponseEntity.created(locationUri).body(created);

// 204 No Content — body is empty, Void signals "intentionally no body"
ResponseEntity.noContent().build();

// 404 Not Found — no body
ResponseEntity.notFound().build();

// 400 Bad Request with custom error body
ResponseEntity.badRequest().body(errorResponse);

// Any status + custom headers
ResponseEntity
    .status(HttpStatus.ACCEPTED)          // 202
    .header("X-Job-ID", jobId)
    .body(jobStatus);

// No body at all (Void generic type)
ResponseEntity<Void> response = ResponseEntity.noContent().build();
```

The key rule: use `.build()` when there's no body, use `.body(value)` when there is.

---

### Real-world examples

#### GET — fetch a resource that may or may not exist

```java
@GetMapping("/{id}")
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(ResponseEntity::ok)                    // 200 OK with body
        .orElse(ResponseEntity.notFound().build()); // 404 No body
}
```

`userService.findById()` returns an `Optional<UserResponse>`. `.map()` converts it to a `ResponseEntity` only if present. This is the idiomatic pattern — no if/else, no exceptions for the not-found case.

---

#### POST — create a resource and tell the client where it lives

```java
@PostMapping
public ResponseEntity<UserResponse> createUser(@RequestBody CreateUserRequest req) {
    UserResponse created = userService.create(req);

    // Build the URL of the newly created resource: /api/v1/users/42
    URI location = ServletUriComponentsBuilder
        .fromCurrentRequest()         // starts from the current request URL
        .path("/{id}")                // appends /{id}
        .buildAndExpand(created.getId())
        .toUri();

    return ResponseEntity
        .created(location)            // 201 Created + Location: /api/v1/users/42
        .body(created);
}
```

Why bother with the `Location` header? REST clients and API gateways use it to know where the resource now lives — so they can fetch it without hardcoding a URL. This is standard REST convention (RFC 7231).

---

#### PATCH — update that may succeed or fail silently

```java
@PatchMapping("/{id}/status")
public ResponseEntity<Void> updateStatus(@PathVariable Long id, @RequestBody StatusUpdate req) {
    boolean updated = userService.updateStatus(id, req.getStatus());
    return updated
        ? ResponseEntity.noContent().build()        // 204 — updated, no body needed
        : ResponseEntity.notFound().build();        // 404 — nothing to update
}
```

`ResponseEntity<Void>` is the correct type when you intentionally return no body. `Void` (capital V) is a Java convention for "this generic slot is empty by design."

---

#### Adding custom response headers

Real scenario: an async export job — client kicks it off, gets a job ID back, polls later.

```java
@PostMapping("/exports")
public ResponseEntity<ExportJobResponse> startExport(@RequestBody ExportRequest req) {
    ExportJob job = exportService.submit(req);

    URI statusUrl = ServletUriComponentsBuilder
        .fromCurrentRequest()
        .path("/{jobId}")
        .buildAndExpand(job.getId())
        .toUri();

    return ResponseEntity
        .accepted()                              // 202 Accepted (processing async)
        .location(statusUrl)                     // Location: /exports/abc-123
        .header("Retry-After", "5")              // hint: poll in 5 seconds
        .header("X-Job-ID", job.getId())
        .body(new ExportJobResponse(job));
}
```

The client receives `202 Accepted` with enough headers to poll for status. All of this is impossible with a plain return type.

---

#### Conditional responses — ETags and caching

```java
@GetMapping("/{id}")
public ResponseEntity<UserResponse> getUser(
        @PathVariable Long id,
        @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {

    UserResponse user = userService.findById(id)
        .orElseThrow(() -> new UserNotFoundException(id));

    String etag = "\"" + user.getVersion() + "\""; // version from DB row

    if (etag.equals(ifNoneMatch)) {
        return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build(); // 304 — client cache is fresh
    }

    return ResponseEntity.ok()
        .eTag(etag)
        .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
        .body(user);
}
```

This is how APIs reduce bandwidth: the second time a client fetches the same user, they send `If-None-Match: "3"`. If the user hasn't changed (version is still 3), we return `304` with no body — the client uses its cached copy.

---

### `ResponseEntity` vs `@ResponseStatus` — when to use which

| Situation | Prefer |
|---|---|
| Status code is always the same for this method | `@ResponseStatus(HttpStatus.CREATED)` on the method |
| Status code depends on runtime data (found/not found) | `ResponseEntity` |
| Need to set response headers dynamically | `ResponseEntity` |
| Simple CRUD that always succeeds | Plain return type (Spring defaults to 200) |
| POST that always creates | `@ResponseStatus(HttpStatus.CREATED)` |

```java
// Simple — @ResponseStatus is cleaner here
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public UserResponse createUser(@RequestBody CreateUserRequest req) {
    return userService.create(req); // always 201, always has body, no dynamic headers
}

// Complex — ResponseEntity needed here
@PostMapping
public ResponseEntity<UserResponse> createUserWithLocation(@RequestBody CreateUserRequest req) {
    // need Location header → must use ResponseEntity
}
```

---

### Common mistakes

**1. Returning `ResponseEntity<Object>` to "handle multiple types"**

```java
// BAD — loses type safety, confuses Swagger, hard to test
public ResponseEntity<Object> getUser(@PathVariable Long id) {
    if (notFound) return ResponseEntity.notFound().build();
    return ResponseEntity.ok(user);         // Object type loses UserResponse info
}

// GOOD — use a typed wrapper or throw a typed exception
public ResponseEntity<UserResponse> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(ResponseEntity::ok)
        .orElse(ResponseEntity.notFound().build());
}
```

**2. Forgetting `.build()` on no-body responses**

```java
// BAD — compile error or empty body serialized as null
return ResponseEntity.notFound();             // returns BodyBuilder, not ResponseEntity

// GOOD
return ResponseEntity.notFound().build();     // returns ResponseEntity<T>
```

**3. Not setting `Location` header on 201 responses**

```java
// BAD — technically works but violates REST convention
return ResponseEntity.status(201).body(created);

// GOOD — clients know where the resource lives
return ResponseEntity.created(locationUri).body(created);
```

---

### Practice Questions

**Q1: What is the difference between `ResponseEntity.ok(body)` and returning `body` directly from a controller?**

Returning `body` directly tells Spring "use the default status (200) and serialize this." `ResponseEntity.ok(body)` does the same thing but gives you a handle to add headers or change the status later. Functionally identical for simple cases, but `ResponseEntity` is required the moment you need anything beyond a 200 with a body.

---

**Q2: Why is `ResponseEntity<Void>` used instead of `ResponseEntity<String>` with an empty string for 204 responses?**

`Void` (capital V) is a Java signal meaning "this type intentionally has no value." Using `String` would imply a body is possible but happens to be empty. Using `Void` makes the intent explicit in the type signature — any reader of the code immediately knows this endpoint never returns a body. It also prevents accidental body serialization.

---

**Q3: A client sends a `POST /users` and your service creates the user. What HTTP status and headers should your response include, and why?**

- **Status: 201 Created** — distinguishes "resource was created" from "request was processed" (200). APIs that return 200 for creates violate REST semantics.
- **Location header** — set to the URL of the newly created resource (e.g., `/api/v1/users/42`). This tells the client where to find what they just created without requiring a second lookup.
- **Body** — the created resource, so the client doesn't need to immediately GET it.

```java
return ResponseEntity.created(locationUri).body(created);
```

---

**Q4: You need to return a `200 OK` or `202 Accepted` depending on whether processing happened synchronously or was queued. How would you model this with `ResponseEntity`?**

```java
@PostMapping("/reports")
public ResponseEntity<ReportResponse> generateReport(@RequestBody ReportRequest req) {
    if (req.isSmall()) {
        // Processed immediately
        ReportResponse report = reportService.generateSync(req);
        return ResponseEntity.ok(report);                    // 200 — done now
    } else {
        // Queued for async processing
        ReportJob job = reportService.queueAsync(req);
        URI statusUrl = buildStatusUrl(job.getId());
        return ResponseEntity.accepted()                     // 202 — not done yet
            .location(statusUrl)
            .body(new ReportResponse(job));
    }
}
```

This is impossible with `@ResponseStatus` since the code is determined at runtime.

---

**Q5: What happens if you return `ResponseEntity.notFound().build()` but the generic type is `ResponseEntity<UserResponse>`? Is this a compile error?**

No compile error. `notFound().build()` returns `ResponseEntity<Object>` internally, but Java's type inference resolves it to match the declared return type `ResponseEntity<UserResponse>` at the call site. At runtime, there is no body, so the generic type doesn't matter. The type parameter only matters when there is an actual body to serialize.

---

## 3.5 Global Exception Handling with `@ControllerAdvice`

Instead of try-catching in every controller, centralize all exception handling:

### Error Response DTO

```java
public class ErrorResponse {
    private int status;
    private String message;
    private Instant timestamp;
    private Map<String, String> fieldErrors; // for validation failures

    // constructors, getters
}
```

### `@ControllerAdvice` Handler

```java
@RestControllerAdvice  // @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    // Custom business exception
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, ex.getMessage(), Instant.now()));
    }

    // Validation failure — thrown when @Valid fails
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationError(
            MethodArgumentNotValidException ex) {
        Map<String, String> fieldErrors = ex.getBindingResult()
            .getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
        ErrorResponse error = new ErrorResponse(400, "Validation failed", Instant.now());
        error.setFieldErrors(fieldErrors);
        return ResponseEntity.badRequest().body(error);
    }

    // Catch-all for unexpected exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unhandled exception", ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(500, "Internal server error", Instant.now()));
    }
}
```

### Custom Exception Pattern

```java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(Long id) {
        super("User not found with id: " + id);
    }
}

// In service:
public UserResponse findById(Long id) {
    return userRepository.findById(id)
        .map(userMapper::toResponse)
        .orElseThrow(() -> new UserNotFoundException(id));
}
```

---

## 3.6 Bean Validation

Add constraints on your request DTOs:

```java
public class CreateUserRequest {

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100)
    private String name;

    @NotBlank
    @Email(message = "Invalid email format")
    private String email;

    @NotNull
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 120)
    private Integer age;

    @Pattern(regexp = "^\\+?[1-9]\\d{7,14}$", message = "Invalid phone number")
    private String phone;

    @Valid  // triggers validation on nested object
    @NotNull
    private AddressRequest address;
}
```

Trigger validation with `@Valid` on the parameter:

```java
@PostMapping
public ResponseEntity<UserResponse> create(@Valid @RequestBody CreateUserRequest req) {
    // if validation fails, MethodArgumentNotValidException is thrown before this runs
    return ResponseEntity.ok(userService.create(req));
}
```

### Custom Validator

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    @Autowired
    private UserRepository userRepository;

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return email == null || !userRepository.existsByEmail(email);
    }
}
```

---

## 3.7 Filters, Interceptors, and AOP Compared

### Filters (Servlet Level — Before Spring)

```java
@Component
@Order(1)  // execution order among filters
public class CorrelationIdFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {
        String correlationId = request.getHeader("X-Correlation-ID");
        if (correlationId == null) correlationId = UUID.randomUUID().toString();

        MDC.put("correlationId", correlationId);  // available in all logs for this request
        response.setHeader("X-Correlation-ID", correlationId);

        try {
            chain.doFilter(request, response); // pass to next filter / servlet
        } finally {
            MDC.remove("correlationId");
        }
    }
}
```

### Interceptors (Spring MVC Level — After DispatcherServlet)

```java
@Component
public class AuditInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        log.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        return true; // return false to abort request
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler, Exception ex) {
        log.info("Response: {}", response.getStatus());
    }
}

// Register the interceptor:
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private AuditInterceptor auditInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(auditInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/health");
    }
}
```

### When to Use What

| Scenario | Use |
|---|---|
| Auth token validation (JWT parsing) | Filter — needs to run before Spring Security |
| CORS headers | Filter — framework handles this |
| Audit logging of all endpoints | Interceptor — access to handler method details |
| Request timing for specific controllers | Interceptor |
| Logging method entry/exit for services | AOP |
| Transaction management | AOP (Spring does this internally) |

---

## 3.8 CORS Configuration

```java
@Configuration
public class CorsConfig {

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

Or per-controller:
```java
@CrossOrigin(origins = "https://app.example.com", maxAge = 3600)
@RestController
public class UserController { ... }
```

---

## Tricky Interview Questions

**Q: What happens if you annotate a method with `@GetMapping` but the client sends a POST? What's the HTTP status?**

Spring returns **405 Method Not Allowed** (handled by `DefaultHandlerExceptionResolver`). The response includes an `Allow` header listing supported methods.

---

**Q: You have a `@ControllerAdvice` that handles `Exception.class`. A controller throws a `NullPointerException`. Does the advice catch it?**

Yes — `NullPointerException` is a subclass of `RuntimeException` which is a subclass of `Exception`. The `@ExceptionHandler(Exception.class)` acts as a catch-all. However, if there are more specific handlers (e.g., `@ExceptionHandler(RuntimeException.class)`), Spring uses the most specific match.

---

**Q: What's the difference between `@RequestBody` and `@ModelAttribute`?**

- `@RequestBody` — reads from the HTTP request body; works with JSON/XML; uses `HttpMessageConverter` (Jackson)
- `@ModelAttribute` — binds form data or URL query parameters to a Java object; works with `application/x-www-form-urlencoded` (HTML forms)

```java
// @RequestBody — for REST APIs with JSON body
@PostMapping("/api/users")
public UserResponse create(@RequestBody CreateUserRequest req) { ... }

// @ModelAttribute — for form submissions
@PostMapping("/form/register")
public String register(@ModelAttribute RegistrationForm form) { ... }
```

---

**Q: You want to return a `404` status without throwing an exception. How?**

Two ways:

```java
// Option 1: ResponseEntity
return ResponseEntity.notFound().build();

// Option 2: @ResponseStatus on the exception class
@ResponseStatus(HttpStatus.NOT_FOUND)
public class UserNotFoundException extends RuntimeException { ... }
// Spring auto-converts this exception to a 404 response
```

---

**Q: A filter runs before Spring Security. Your auth filter sets `SecurityContextHolder`. Why does this work?**

Filters run before `DispatcherServlet` and before Spring Security's filter chain in the Servlet pipeline. But Spring Security itself is a filter — `DelegatingFilterProxy` wraps the Spring Security filter chain and also runs before `DispatcherServlet`. The order is:

```
Your Filters (if registered with lower @Order)
→ Spring Security Filter Chain (DelegatingFilterProxy)
  → UsernamePasswordAuthenticationFilter
  → JwtAuthenticationFilter (custom)
  → ...
→ DispatcherServlet
→ Your Interceptors
→ Controller
```

If you write a JWT filter that extends `OncePerRequestFilter` and is added to the Spring Security filter chain, you can set `SecurityContextHolder` and downstream security checks will see it.

---

**Q: How would you version a REST API?**

Three common strategies:

```java
// 1. URL versioning (most common)
@RequestMapping("/api/v1/users")
@RequestMapping("/api/v2/users")

// 2. Header versioning
@GetMapping(value = "/users", headers = "X-API-Version=2")

// 3. Content negotiation (Accept header)
@GetMapping(value = "/users", produces = "application/vnd.company.api-v2+json")
```

URL versioning is most common because it's explicit and easy to test in a browser. Header/content-type versioning is cleaner but harder to cache.
