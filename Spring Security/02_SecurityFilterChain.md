# Security Filter Chain

## What is the Filter Chain?

Every HTTP request to your Spring Boot app passes through a **chain of filters** before reaching your controller. Each filter has one job. Together they form the Security Filter Chain.

Think of it like a production pipeline:
```
Request → [Filter 1] → [Filter 2] → [Filter 3] → ... → [Your Controller]
```

If any filter decides the request is invalid (no token, wrong role, etc.), it short-circuits the chain and sends a response immediately — your controller never sees the request.

---

## The Architecture: DelegatingFilterProxy and FilterChainProxy

Your app is a Servlet application. Filters in Servlet world know nothing about Spring beans. Spring Security bridges this gap:

```
Servlet Container
    |
    v
DelegatingFilterProxy
// DelegatingFilterProxy — a standard Servlet Filter registered with the container
// Its only job: delegate to a Spring-managed bean named "springSecurityFilterChain"
// This is how the Servlet world calls into the Spring bean world
    |
    v
FilterChainProxy (the actual bean named "springSecurityFilterChain")
// FilterChainProxy — Spring Security's main entry point
// Holds a list of SecurityFilterChain objects
// Matches the incoming request to the right chain
    |
    v
SecurityFilterChain (your configuration)
// SecurityFilterChain — the chain you configure in your SecurityConfig
// Contains an ordered list of Security filters
    |
    v
Individual Filters (run in order)
```

---

## Key Filters and What They Do

These are the filters Spring Security registers. You don't write them — Spring adds them based on your configuration. But you MUST know what they do:

### Order matters. Here's the default order (simplified):

```
1.  DisableEncodeUrlFilter
2.  WebAsyncManagerIntegrationFilter
3.  SecurityContextHolderFilter (or SecurityContextPersistenceFilter in older versions)
4.  HeaderWriterFilter
5.  CorsFilter
6.  CsrfFilter
7.  LogoutFilter
8.  UsernamePasswordAuthenticationFilter
9.  DefaultLoginPageGeneratingFilter
10. DefaultLogoutPageGeneratingFilter
11. BasicAuthenticationFilter
12. RequestCacheAwareFilter
13. SecurityContextHolderAwareRequestFilter
14. AnonymousAuthenticationFilter
15. SessionManagementFilter
16. ExceptionTranslationFilter
17. FilterSecurityInterceptor (or AuthorizationFilter in newer versions)
```

### The Most Important Filters Explained:

#### SecurityContextHolderFilter
```
Job: Loads the SecurityContext from the session (or request) at the start of each request,
     and clears it at the end.

Before your code runs:  SecurityContextHolder is populated with the current user
After your code runs:   SecurityContextHolder is cleared (prevents memory leaks)
```

#### UsernamePasswordAuthenticationFilter
```
Job: Handles the default login form submission (POST /login)
Triggers: Only on POST to /login (configurable)

What it does:
1. Extracts username and password from request params
2. Creates UsernamePasswordAuthenticationToken
3. Passes to AuthenticationManager
4. On success: stores Authentication in SecurityContext, redirects
5. On failure: calls AuthenticationFailureHandler
```

#### BasicAuthenticationFilter
```
Job: Handles HTTP Basic Auth (Authorization: Basic base64(user:pass) header)
Triggers: When an Authorization header with "Basic" prefix is present

What it does:
1. Decodes the Base64 header
2. Creates UsernamePasswordAuthenticationToken
3. Passes to AuthenticationManager
4. On success: stores in SecurityContext and continues chain

Note: Basic auth sends credentials on EVERY request (stateless by nature)
```

#### ExceptionTranslationFilter
```
Job: Catches security exceptions from the rest of the chain and translates them to HTTP responses

Catches:
- AuthenticationException → calls AuthenticationEntryPoint → 401 response
- AccessDeniedException   → calls AccessDeniedHandler      → 403 response

This is the bridge between Java exceptions and HTTP error responses.
```

#### AnonymousAuthenticationFilter
```
Job: If the request reaches this point with NO authentication,
     it creates an anonymous Authentication token.

Why: So downstream code never has to check for null authentication.
     Even unauthenticated users have an Authentication object with role ROLE_ANONYMOUS.
```

#### FilterSecurityInterceptor / AuthorizationFilter
```
Job: The final gatekeeper — checks if the authenticated user has permission for this URL.
Runs: Right before the request reaches the DispatcherServlet.

What it does:
1. Looks up access rules for the current URL (from your authorizeHttpRequests config)
2. Checks if the current Authentication satisfies those rules
3. Allows or throws AccessDeniedException
```

---

## Configuring the Filter Chain

### Multiple Security Filter Chains

You can have multiple `SecurityFilterChain` beans for different URL groups. This is common in production:

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    @Order(1)
    // @Order(1) — this chain is checked first (lower number = higher priority)
    // Spring Security checks chains in order and uses the FIRST one that matches
    public SecurityFilterChain apiSecurityChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            // securityMatcher() — this chain ONLY applies to requests matching /api/**
            // If the URL doesn't match, Spring moves to the next chain
            
            .csrf(csrf -> csrf.disable())
            // APIs are stateless, no CSRF needed
            
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // STATELESS — no session cookie for API requests
            
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
            // addFilterBefore(filter, position) — insert custom filter BEFORE the specified filter
            // Our JWT filter runs before the default username/password filter
        
        return http.build();
    }

    @Bean
    @Order(2)
    // @Order(2) — this chain is checked second, catches all non-/api/** requests
    public SecurityFilterChain webSecurityChain(HttpSecurity http) throws Exception {
        http
            // No securityMatcher — matches everything not matched by chain 1
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                // formLogin() — enables the standard form-based login
                .loginPage("/login")
                // loginPage() — URL of your custom login page
                .defaultSuccessUrl("/dashboard", true)
                // defaultSuccessUrl() — where to go after successful login
                // true = always redirect here, even if user was going somewhere else
                .failureUrl("/login?error=true")
                // failureUrl() — where to go after failed login attempt
                .permitAll()
            )
            .logout(logout -> logout
                // logout() — configures logout behavior
                .logoutUrl("/logout")
                // logoutUrl() — URL that triggers logout (POST by default)
                .logoutSuccessUrl("/login?logout=true")
                // logoutSuccessUrl() — redirect here after logout
                .invalidateHttpSession(true)
                // invalidateHttpSession() — destroys the HTTP session on logout
                .clearAuthentication(true)
                // clearAuthentication() — clears the SecurityContext on logout
                .deleteCookies("JSESSIONID")
                // deleteCookies() — removes the session cookie from the browser
            );
        
        return http.build();
    }
}
```

### Adding Your Own Custom Filter

This is the most common thing you'll do in a real project — adding a JWT filter:

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    // OncePerRequestFilter — base class that guarantees the filter runs exactly once per request
    // Even if the request is forwarded/dispatched internally, it only runs once

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtTokenProvider jwtTokenProvider,
                                    UserDetailsService userDetailsService) {
        this.jwtTokenProvider = jwtTokenProvider;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
                                     throws ServletException, IOException {
        // doFilterInternal() — the method you implement; runs for each request
        // HttpServletRequest  — the incoming HTTP request
        // HttpServletResponse — the outgoing HTTP response  
        // FilterChain         — call filterChain.doFilter() to continue the chain

        String authHeader = request.getHeader("Authorization");
        // getHeader() — reads the HTTP header value by name

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            // No token present — skip this filter, let the chain continue
            // The AnonymousAuthenticationFilter will set anonymous auth later
            filterChain.doFilter(request, response);
            // doFilter() — pass control to the next filter in the chain
            return;
        }

        String token = authHeader.substring(7);
        // substring(7) — strips "Bearer " prefix (7 characters)

        try {
            if (jwtTokenProvider.validateToken(token)) {
                // validateToken() — your method to verify signature, expiry, etc.

                String username = jwtTokenProvider.getUsernameFromToken(token);
                // getUsernameFromToken() — your method to extract username from JWT claims

                UserDetails userDetails = userDetailsService.loadUserByUsername(username);
                // loadUserByUsername() — loads full user details (including roles) from DB

                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,       // principal — the UserDetails object
                        null,              // credentials — null because JWT is already verified
                        userDetails.getAuthorities()
                        // getAuthorities() — the user's roles/permissions
                    );
                // UsernamePasswordAuthenticationToken(principal, credentials, authorities)
                // 3-arg constructor marks the token as authenticated (isAuthenticated() == true)
                // 2-arg constructor (no authorities) marks it as NOT yet authenticated

                authentication.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                    // WebAuthenticationDetailsSource — captures extra request details (IP, session ID)
                    // Useful for auditing: "user X authenticated from IP Y"
                );

                SecurityContextHolder.getContext().setAuthentication(authentication);
                // setAuthentication() — stores the verified Authentication in the current thread's context
                // This is what makes the user "logged in" for the duration of this request
            }
        } catch (JwtException ex) {
            // Token is invalid/expired — clear any partial auth and send 401
            SecurityContextHolder.clearContext();
            // clearContext() — removes Authentication from the current thread
            response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Invalid token");
            // sendError() — writes an HTTP error response and stops filter chain
            return;
        }

        filterChain.doFilter(request, response);
        // Continue the chain — now with (or without) authentication set
    }

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        // shouldNotFilter() — return true to SKIP this filter for certain requests
        // This is cleaner than checking URL patterns inside doFilterInternal
        String path = request.getServletPath();
        return path.startsWith("/api/auth/") || path.startsWith("/public/");
    }
}
```

Registering the filter in your config:

```java
@Bean
public SecurityFilterChain apiSecurityChain(HttpSecurity http,
                                              JwtAuthenticationFilter jwtFilter) throws Exception {
    http
        .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
        // addFilterBefore(myFilter, existingFilter.class)
        // — inserts myFilter immediately BEFORE the specified existing filter
        // Common positions:
        //   addFilterBefore  → runs BEFORE the specified filter
        //   addFilterAfter   → runs AFTER the specified filter
        //   addFilterAt      → runs AT THE SAME POSITION as the specified filter
        // JWT filter before UsernamePasswordAuthenticationFilter means JWT auth
        // runs before Spring tries form login — correct order for JWT APIs
        ;
    
    return http.build();
}
```

### How Spring Injects the Filter into `securityFilterChain`

You might wonder: *"The method signature has `JwtAuthenticationFilter jwtFilter` as a parameter — who passes it in? I never call `securityFilterChain()` anywhere."*

Spring calls it. Because the method is annotated `@Bean`, Spring intercepts it at startup, resolves all parameters from its application context, then invokes the method and stores the returned `SecurityFilterChain` bean.

Think of Spring's context as a registry keyed by type:

```
Application Context Registry (built at startup):
  JwtAuthenticationFilter  →  [instance returned by jwtAuthFilter() @Bean method]
  HttpSecurity             →  [created internally by Spring Security]
  DataSource               →  [created from your DB config]
  ...
```

When Spring needs to call `securityFilterChain(...)`, it does the equivalent of:

```java
// What Spring does internally — you never write this yourself
JwtAuthenticationFilter filter = context.getBean(JwtAuthenticationFilter.class);
HttpSecurity http = context.getBean(HttpSecurity.class);

securityFilterChain(http, filter);  // Spring calls your @Bean method with resolved args
```

**Spring matches by type, not by parameter name.** The parameter could be named `myFilter` or `xyz` and injection would still work — it looks for a bean of type `JwtAuthenticationFilter.class` in the context.

If no matching bean exists (e.g., you forgot `@Bean` on `jwtAuthFilter()`), startup fails immediately:

```
NoSuchBeanDefinitionException: No qualifying bean of type
'com.example.security.JwtAuthenticationFilter' available
```

---

## Debugging the Filter Chain

To see exactly which filters are registered and in what order, enable debug logging:

```yaml
# application.yml
logging:
  level:
    org.springframework.security: DEBUG
    org.springframework.security.web.FilterChainProxy: DEBUG
```

This prints something like:
```
Security filter chain: [
  DisableEncodeUrlFilter
  WebAsyncManagerIntegrationFilter
  SecurityContextHolderFilter
  HeaderWriterFilter
  JwtAuthenticationFilter          <-- your custom filter appears here
  UsernamePasswordAuthenticationFilter
  ...
]
```

---

## Common Production Gotchas

### Gotcha 1: Filter registered as Spring bean AND added to chain

There are **two separate systems** that can register a filter in a Spring Boot app, and using `@Component` on a filter accidentally activates both.

**System 1 — Servlet Container (Tomcat):** Spring Boot's auto-configuration scans the application context. If it finds any bean implementing `javax.servlet.Filter`, it automatically registers it directly with Tomcat. Every HTTP request hits these filters before Spring Security is involved.

**System 2 — Spring Security Filter Chain:** When you call `http.addFilterBefore(jwtFilter, ...)` in your `SecurityFilterChain` config, you're manually inserting the filter into Spring Security's internal chain.

```
HTTP Request
     ↓
[ Tomcat's Filter Pipeline ]   ← System 1 (auto-registration of @Component filters)
     ↓
[ Spring Security Filter Chain (runs inside one Tomcat filter) ]   ← System 2
     ↓
[ Your Controller ]
```

If you annotate your filter with `@Component` AND add it via `addFilterBefore`, both systems register it — your filter runs **twice** per request:

```java
@Component  // <-- triggers System 1: Spring Boot auto-registers this with Tomcat
public class JwtAuthenticationFilter extends OncePerRequestFilter { ... }

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class); // System 2
    return http.build();
}
```

```
Incoming POST /api/orders  (Authorization: Bearer eyJhbGc...)

1. Tomcat runs JwtAuthenticationFilter    ← System 1 fires (auto-registration)
   → JWT validated, SecurityContext populated
2. Tomcat hits Spring Security's FilterChainProxy
3. Spring Security runs JwtAuthenticationFilter AGAIN  ← System 2 fires
   → JWT processed a second time
4. Request reaches controller
```

In a simple read-only filter, double execution is just wasteful. In real systems it causes actual bugs:
- **Rate limiting filter** — every request is counted twice against the quota
- **Audit logging filter** — every request appears twice in audit trails
- **Token rotation filter** — token is rotated on the first pass; second pass sees the old token and returns 401
- **DB-hit filter** — two database calls per request for no reason

**Fix Option 1 — `FilterRegistrationBean` to opt out of System 1:**

```java
@Bean
public FilterRegistrationBean<JwtAuthenticationFilter> jwtFilterRegistration(
        JwtAuthenticationFilter filter) {
    // FilterRegistrationBean — wrapper to control how a filter registers with Tomcat
    FilterRegistrationBean<JwtAuthenticationFilter> registration = new FilterRegistrationBean<>(filter);
    registration.setEnabled(false);
    // setEnabled(false) — tells Tomcat NOT to register this filter directly
    // The filter only runs when explicitly added to the Spring Security chain
    return registration;
}
```

**Fix Option 2 (preferred) — Don't use `@Component` at all. Declare the filter as a `@Bean` in your config class:**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public JwtAuthenticationFilter jwtAuthFilter() {
        return new JwtAuthenticationFilter(jwtUtil);
        // No @Component on the filter class → Spring Boot never auto-registers it with Tomcat
        // Only System 2 (addFilterBefore) controls where it runs
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http,
                                                    JwtAuthenticationFilter jwtAuthFilter)
            throws Exception {
        http.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

This is the preferred pattern in production — intent is explicit, no hidden auto-registration behavior.

**When direct Tomcat registration (System 1) is intentional:**

If a filter should run *outside* Spring Security entirely — CORS, request ID stamping, health-check bypass — you do want `@Component` alone without `addFilterBefore`. The rule:
- Needs `SecurityContext`, participates in auth → Spring Security chain only (System 2)
- Pure infrastructure concern (tracing, CORS) → Tomcat-level is fine (System 1)

### Gotcha 2: Wrong filter position breaks auth

```java
// Bad: Adding JWT filter AFTER UsernamePasswordAuthenticationFilter
// The UsernamePasswordAuthenticationFilter might reject the request first
http.addFilterAfter(jwtFilter, UsernamePasswordAuthenticationFilter.class); // Wrong

// Good: JWT filter runs BEFORE, sets auth, then chain continues normally
http.addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class); // Correct
```

### Gotcha 3: Forgetting `filterChain.doFilter()` at the end

```java
// If you forget to call doFilter(), the request stops at your filter.
// Your controller never runs. You get a blank response with no error.

protected void doFilterInternal(...) {
    // ... your logic ...
    
    // MUST call this to continue the chain:
    filterChain.doFilter(request, response); // Don't forget this!
}
```
