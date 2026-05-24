# Spring Security Filter Chain — Internal Implementation

## Prerequisite: How Tomcat Forwards a Request to Spring Boot

Understanding this makes Spring Security's design obvious rather than magical.

### Without Spring Security — the full journey

```
Client
  │  HTTP request (e.g. GET /api/orders)
  ▼
Tomcat Connector (Coyote)
  │  Parses raw TCP/HTTP bytes
  │  Creates HttpServletRequest + HttpServletResponse objects
  ▼
Servlet Filter Chain  ← runs first, can short-circuit
  │  CharacterEncodingFilter
  │  RequestContextFilter
  │  ... any registered filters ...
  ▼
DispatcherServlet  ← Spring's front controller (a Servlet, not a Filter)
  │  HandlerMapping  →  "GET /api/orders maps to OrderController.getOrders()"
  │  HandlerAdapter  →  invokes the method, resolves @RequestBody etc.
  │  MessageConverter →  serializes return value to JSON
  ▼
HttpServletResponse → Tomcat sends bytes back to client
```

Key point: `DispatcherServlet` is registered with Tomcat as a **Servlet** mapped to `"/"` — it is **never** part of the filter chain. Filters and Servlets are two different layers in the Servlet spec:

| Layer | Runs | Can short-circuit? |
|---|---|---|
| Servlet Filters | Before the Servlet | Yes — don't call `chain.doFilter()` |
| Servlet (`DispatcherServlet`) | After all filters pass | N/A |

### Common misconception

> "When Spring Security is added, `DispatcherServlet` is no longer registered with Tomcat."

**Wrong.** `DispatcherServlet` is always registered with Tomcat as a Servlet. What Spring Security does is **insert `FilterChainProxy` into the Servlet filter chain**, so security checks run before the request ever reaches `DispatcherServlet`:

```
Without Security:
  Filters (Encoding, etc.)  →  DispatcherServlet  →  @Controller

With Security:
  Filters (Encoding, etc.)  →  FilterChainProxy  →  DispatcherServlet  →  @Controller
                                      │
                               JWT checks, auth, authz
                               (blocks here if failed — controller never reached)
```

The correct way to say it: **"Spring Security adds `FilterChainProxy` into the Servlet filter chain. If security checks fail, the request is short-circuited before reaching `DispatcherServlet`. But `DispatcherServlet` itself is always a Servlet — it is not, and never was, part of the filter chain."**

---

## The 4 Key Classes/Interfaces

```
Filter (jakarta.servlet)
    └── DelegatingFilterProxy        [org.springframework.web.filter]
    └── FilterChainProxy             [org.springframework.security.web]
    └── DefaultSecurityFilterChain   [org.springframework.security.web]  ← implements SecurityFilterChain
```

---

### 1. `Filter` — the foundation

Every component in Spring Security implements this Servlet interface:

```java
public interface Filter {
    void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException;
}
```

The entire security chain is just nested `doFilter()` calls. No magic — just one filter calling `chain.doFilter()` to pass control to the next.

---

### 2. `DelegatingFilterProxy` — Servlet → Spring bridge

Registered with Tomcat at startup. Tomcat knows nothing about Spring beans, so this class acts as a named placeholder.

```java
public class DelegatingFilterProxy extends GenericFilterBean {

    private String targetBeanName; // = "springSecurityFilterChain"
    private volatile Filter delegate; // lazily resolved on first request

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        if (this.delegate == null) {
            // First request: fetch the real bean from Spring's ApplicationContext
            this.delegate = applicationContext.getBean("springSecurityFilterChain", Filter.class);
        }
        // Hand off to FilterChainProxy — this is the only thing it does
        this.delegate.doFilter(request, response, chain);
    }
}
```

**Interview line:** "DelegatingFilterProxy is registered with Tomcat but delegates all work to a Spring-managed bean. It's just a bridge between the Servlet world and the Spring bean world."

---

### 3. `FilterChainProxy` — Spring Security's main entry point

This IS the `"springSecurityFilterChain"` bean. It holds ALL your `SecurityFilterChain` beans.

```java
public class FilterChainProxy extends GenericFilterBean {

    private List<SecurityFilterChain> filterChains; // all your @Bean SecurityFilterChain configs

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        // 1. Find the first SecurityFilterChain whose URL pattern matches this request
        List<Filter> filters = getFilters((HttpServletRequest) request);

        if (filters == null || filters.isEmpty()) {
            chain.doFilter(request, response); // no security chain matched, pass through
            return;
        }

        // 2. Wrap the matched filters in a VirtualFilterChain and execute them
        new VirtualFilterChain(chain, filters).doFilter(request, response);
    }

    private List<Filter> getFilters(HttpServletRequest request) {
        for (SecurityFilterChain chain : filterChains) {
            if (chain.matches(request)) {   // checks /api/** or /web/** etc.
                return chain.getFilters();  // returns on FIRST match — @Order matters
            }
        }
        return null;
    }
}
```

**`VirtualFilterChain`** — the inner class that executes filters sequentially:

```java
private static final class VirtualFilterChain implements FilterChain {
    private final FilterChain originalChain; // Tomcat's chain (leads to DispatcherServlet)
    private final List<Filter> filters;      // your security filters
    private int currentPosition = 0;

    public void doFilter(ServletRequest request, ServletResponse response) {
        if (currentPosition == filters.size()) {
            originalChain.doFilter(request, response); // all security filters done → DispatcherServlet
            return;
        }
        Filter next = filters.get(currentPosition++);
        next.doFilter(request, response, this); // `this` is passed as the chain
        // when the filter calls chain.doFilter(), it comes back here → increments position → next filter
    }
}
```

**Interview line:** "FilterChainProxy picks the right SecurityFilterChain for the request URL, then runs all its filters through VirtualFilterChain — which is just a position counter over a list."

---

### 4. `SecurityFilterChain` — the interface your config produces

```java
public interface SecurityFilterChain {
    boolean matches(HttpServletRequest request); // does this chain handle this URL?
    List<Filter> getFilters();                   // the ordered list of filters
}
```

The concrete implementation built by `http.build()`:

```java
public final class DefaultSecurityFilterChain implements SecurityFilterChain {
    private final RequestMatcher requestMatcher; // e.g. AntPathRequestMatcher("/api/**")
    private final List<Filter> filters;          // ordered list built by HttpSecurity

    public boolean matches(HttpServletRequest request) {
        return requestMatcher.matches(request);
    }
    public List<Filter> getFilters() { return filters; }
}
```

---

### 5. How `http.build()` wires it all together

```java
// WebSecurityConfiguration (Spring's auto-config)
@Bean(name = "springSecurityFilterChain")   // exact name DelegatingFilterProxy looks for
public Filter springSecurityFilterChain() {
    List<SecurityFilterChain> chains = // collects all your @Bean SecurityFilterChain, sorted by @Order
    return new FilterChainProxy(chains);
}

// Your SecurityConfig
@Bean
public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    http.securityMatcher("/api/**")
        .authorizeHttpRequests(...)
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
    
    return http.build();
    // http.build() → sorts all configured filters by order → new DefaultSecurityFilterChain(matcher, filters)
}
```

---

## Complete Request Lifecycle Example

**Scenario:** User calls `GET /api/orders` with a JWT token.

```
CLIENT
  │
  │  GET /api/orders
  │  Authorization: Bearer eyJhbGci...
  │
  ▼
TOMCAT (Servlet Container)
  │  Receives the request, runs its registered Filters in order
  │  Hits DelegatingFilterProxy (registered at startup)
  │
  ▼
DelegatingFilterProxy.doFilter()
  │  Resolves "springSecurityFilterChain" bean from ApplicationContext (cached after first call)
  │  Calls FilterChainProxy.doFilter()
  │
  ▼
FilterChainProxy.doFilter()
  │  Runs HttpFirewall.getFirewalledRequest() — blocks malformed URLs, path traversal attacks
  │  Iterates SecurityFilterChain list:
  │    Chain 1 (@Order 1): securityMatcher("/api/**") → matches! → returns its List<Filter>
  │  Creates VirtualFilterChain with those filters
  │
  ▼
VirtualFilterChain executes filters in order:

  [1] SecurityContextHolderFilter
  │     Loads SecurityContext from session (or creates empty one for stateless)
  │     Calls chain.doFilter() → next filter
  │
  [2] CsrfFilter
  │     CSRF disabled for /api/** in your config → passes through immediately
  │     Calls chain.doFilter() → next filter
  │
  [3] JwtAuthenticationFilter  ← your custom filter
  │     Reads "Authorization: Bearer eyJhbGci..." header
  │     Validates JWT signature and expiry
  │     Calls userDetailsService.loadUserByUsername("john@example.com")
  │     Creates UsernamePasswordAuthenticationToken(userDetails, null, authorities)
  │     Sets it on SecurityContextHolder.getContext().setAuthentication(token)
  │     Calls chain.doFilter() → next filter
  │
  [4] UsernamePasswordAuthenticationFilter
  │     Checks: is this a POST /login request? No → skips entirely
  │     Calls chain.doFilter() → next filter
  │
  [5] AnonymousAuthenticationFilter
  │     Checks: is SecurityContext already populated? YES (JWT filter did it) → skips
  │     Calls chain.doFilter() → next filter
  │
  [6] ExceptionTranslationFilter
  │     Wraps remaining chain in try/catch
  │     Calls chain.doFilter() → next filter
  │
  [7] AuthorizationFilter (final gatekeeper)
        Reads SecurityContext: user is "john@example.com" with role ROLE_USER
        Checks your rule: .requestMatchers("/api/orders").hasRole("USER") → PASSES
        Calls chain.doFilter() → VirtualFilterChain.currentPosition == filters.size()
        → calls originalChain.doFilter()
  │
  ▼
DispatcherServlet
  │  Maps GET /api/orders → OrderController.getOrders()
  │
  ▼
OrderController.getOrders()
  │  Optionally calls SecurityContextHolder.getContext().getAuthentication()
  │  to get the current user (already set by JwtAuthenticationFilter above)
  │
  ▼
Response flows back up through the same filter chain in reverse
  │  SecurityContextHolderFilter clears the SecurityContext on the way out
  │
  ▼
CLIENT receives 200 OK with order data
```

### What happens on a bad token:

```
[3] JwtAuthenticationFilter
      Validates JWT → signature invalid → throws JwtException
      Calls SecurityContextHolder.clearContext()
      Calls response.sendError(401, "Invalid token")
      returns WITHOUT calling chain.doFilter()
      ↓
Chain is short-circuited — no further filters run, controller never reached
CLIENT receives 401 Unauthorized
```

### What happens with no token at all:

```
[3] JwtAuthenticationFilter
      No "Authorization" header → calls chain.doFilter() immediately (skips)
      ↓
[5] AnonymousAuthenticationFilter
      SecurityContext is empty → sets AnonymousAuthenticationToken (role: ROLE_ANONYMOUS)
      ↓
[7] AuthorizationFilter
      Rule: .anyRequest().authenticated() → ROLE_ANONYMOUS fails
      Throws AccessDeniedException
      ↓
[6] ExceptionTranslationFilter (catches it on the way back up)
      Calls AuthenticationEntryPoint.commence() → response.sendError(401)
CLIENT receives 401 Unauthorized
```

---

## Class Summary for Interview

| Class / Interface | Package | One-line role |
|---|---|---|
| `Filter` | `jakarta.servlet` | Base contract — everything implements this |
| `DelegatingFilterProxy` | `org.springframework.web.filter` | Registered with Tomcat; bridges into Spring beans |
| `FilterChainProxy` | `org.springframework.security.web` | Routes request to the right SecurityFilterChain |
| `SecurityFilterChain` | `org.springframework.security.web` | Interface: matches URL + holds filter list |
| `DefaultSecurityFilterChain` | `org.springframework.security.web` | Concrete impl built by `http.build()` |
| `VirtualFilterChain` | inner class of `FilterChainProxy` | Executes filter list using a position counter |
| `RequestMatcher` | `o.s.s.web.util.matcher` | Strategy for "does this URL match this chain?" |
| `HttpSecurity` | `o.s.s.config.annotation.web.builders` | Fluent builder that produces `DefaultSecurityFilterChain` |

---

## The One-Paragraph Interview Answer

> "Spring Security plugs into the Servlet container via `DelegatingFilterProxy`, which is registered with Tomcat at startup. On the first request, it lazily fetches the `FilterChainProxy` bean — named `springSecurityFilterChain` — from the Spring ApplicationContext. `FilterChainProxy` holds all your `SecurityFilterChain` configurations. For each request, it finds the first chain whose URL matcher matches, then executes that chain's filters in order using an internal `VirtualFilterChain`. Each filter does its job and calls `chain.doFilter()` to pass to the next. When all filters pass, the request reaches `DispatcherServlet` and your controller. If any filter rejects the request — say, `AuthorizationFilter` for a missing role — it throws an exception, `ExceptionTranslationFilter` catches it, and sends the appropriate 401 or 403 response."
