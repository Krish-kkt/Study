# CSRF and CORS

## Part 1: CSRF (Cross-Site Request Forgery)

### What is CSRF?

CSRF is an attack where a malicious website tricks a logged-in user into unknowingly making requests to your application.

**Attack scenario:**
```
1. Alice logs into bank.com → browser stores session cookie
2. Alice visits evil.com (different tab or email link)
3. evil.com has hidden HTML:
   <form action="https://bank.com/transfer" method="POST">
     <input name="amount" value="10000">
     <input name="to" value="attacker_account">
   </form>
   <script>document.forms[0].submit();</script>

4. Browser sends POST to bank.com WITH Alice's session cookie (browsers do this automatically)
5. bank.com thinks Alice requested the transfer — it executes it
```

### Why Does This Work?

Browsers automatically send cookies for any request to a domain, **regardless of which page originated the request**. The bank's server sees a valid session cookie and assumes it's Alice.

### How CSRF Protection Works

Spring Security generates a **CSRF token** — a random, secret value unique to each session. This token must be included in every state-changing request (POST, PUT, DELETE, PATCH).

The attacker on `evil.com` can't read your CSRF token (Same-Origin Policy blocks it), so they can't include it in their forged request.

```
Normal request flow with CSRF:
1. User visits bank.com/transfer (GET)
2. Server responds with form + hidden CSRF token field:
   <input type="hidden" name="_csrf" value="abc123-secret-token">
3. User submits form (POST) with the CSRF token
4. Server verifies token matches stored value → allows request

Forged request from evil.com:
1. evil.com submits POST to bank.com/transfer
2. No CSRF token (evil.com can't read it from another origin)
3. Server rejects: 403 Forbidden
```

### CSRF Configuration in Spring Security

#### For Traditional Web Apps (keep CSRF enabled)

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            // csrf() — configures CSRF protection (enabled by default)
            
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            // CookieCsrfTokenRepository — stores CSRF token in a cookie instead of session
            // withHttpOnlyFalse() — allows JavaScript to read the cookie
            // Use this for SPAs (React, Angular) that need to read the token via JS
            // Default (HttpSessionCsrfTokenRepository) stores token in HTTP session
        );

    return http.build();
}
```

#### In Thymeleaf Templates (auto-handled)

```html
<!-- Thymeleaf with Spring Security auto-includes CSRF token in forms -->
<form th:action="@{/transfer}" method="post">
    <!-- Thymeleaf adds: <input type="hidden" name="_csrf" value="..."> automatically -->
    <input type="number" name="amount">
    <button type="submit">Transfer</button>
</form>
```

#### In React/Angular SPA with Cookie CSRF

```java
// Backend config:
.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    // The CSRF token is set in a cookie named XSRF-TOKEN
    // JavaScript can read it (HttpOnly = false)
)
```

```javascript
// Frontend: Angular HttpClient does this automatically
// For React, read the cookie and send as a header:

function getCsrfToken() {
    return document.cookie
        .split('; ')
        .find(row => row.startsWith('XSRF-TOKEN='))
        ?.split('=')[1];
}

// Include in API calls:
fetch('/api/transfer', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-XSRF-TOKEN': getCsrfToken()  // Spring Security validates this header
    },
    body: JSON.stringify({ amount: 100, to: 'account123' })
});
```

#### For REST APIs — Disable CSRF

Stateless REST APIs that use tokens (JWT, API keys) in headers — not cookies — are **not vulnerable to CSRF**. CSRF exploits the browser's automatic cookie sending. If your auth is header-based, disabling CSRF is correct.

```java
@Bean
public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        // csrf.disable() — disables CSRF protection entirely
        // SAFE when:
        //   ✓ Using JWT in Authorization header (not cookies)
        //   ✓ Using API key in custom header
        //   ✓ Your API is only called by non-browser clients
        // UNSAFE when:
        //   ✗ Your "REST API" uses session cookies for auth
        //   ✗ Your endpoints are called via browser forms
        
        .sessionManagement(session -> 
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
        // Stateless + no cookies = no CSRF vulnerability

    return http.build();
}
```

---

## Part 2: CORS (Cross-Origin Resource Sharing)

### What is CORS?

CORS is a browser security policy that blocks web pages from making requests to a **different origin** than the page was loaded from.

**Origin** = scheme + domain + port. These are all different origins:
```
https://myapp.com          (production)
http://localhost:3000      (React dev server)
https://api.myapp.com      (API subdomain — different!)
https://myapp.com:8080     (different port)
```

**Without CORS:**
```
User is on http://localhost:3000 (React dev server)
React app calls fetch("http://localhost:8080/api/products")
Browser blocks it: "Cross-Origin Request Blocked"
```

CORS headers tell the browser: "It's OK for `http://localhost:3000` to call this API."

### CORS is a Browser Protection

Important: CORS only restricts **browser-based** requests. `curl`, Postman, and server-to-server calls are never blocked by CORS. CORS protects users, not your API.

### How CORS Works

For "simple" GET/POST requests:
```
1. Browser sends request with header: Origin: http://localhost:3000
2. Server responds with: Access-Control-Allow-Origin: http://localhost:3000
3. Browser sees this and allows JavaScript to read the response
4. Without that header: browser receives the response but blocks JS from reading it
```

For "complex" requests (PUT, DELETE, custom headers):
```
1. Browser first sends a "preflight" OPTIONS request
2. Server responds with allowed methods, headers, origins
3. Browser sees approval → sends the actual request
```

### Configuring CORS in Spring Security

#### Global CORS Configuration (Recommended)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            // cors() — enables CORS handling in Spring Security
            // The CorsFilter is added to the filter chain BEFORE auth filters
            // Critical: if CORS filter isn't before auth, preflight OPTIONS requests
            // fail with 401 before reaching the CORS filter
            ;
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        // CorsConfigurationSource — provides CORS configuration for each request

        CorsConfiguration configuration = new CorsConfiguration();
        // CorsConfiguration — holds CORS settings

        configuration.setAllowedOrigins(List.of(
            "http://localhost:3000",        // React dev server
            "https://myapp.com",            // Production frontend
            "https://www.myapp.com"
        ));
        // setAllowedOrigins() — which origins can call this API
        // In production: NEVER use setAllowedOriginPatterns("*") for authenticated endpoints
        // It allows any origin to make requests (CSRF risk for cookie-based auth)

        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        // setAllowedMethods() — which HTTP methods are allowed from the frontend
        // OPTIONS must be included for preflight requests to work

        configuration.setAllowedHeaders(List.of(
            "Authorization",            // JWT token header
            "Content-Type",             // For JSON bodies
            "X-Requested-With",
            "Accept",
            "Origin"
        ));
        // setAllowedHeaders() — which request headers the browser can send
        // If your AJAX sends a header not in this list, the preflight fails

        configuration.setExposedHeaders(List.of(
            "X-Auth-Token",
            "Authorization"
        ));
        // setExposedHeaders() — which RESPONSE headers JavaScript can read
        // By default, JS can only read: Cache-Control, Content-Language, Content-Type, Expires, Last-Modified, Pragma
        // If your API sends custom headers the frontend needs to read, list them here

        configuration.setAllowCredentials(true);
        // setAllowCredentials(true) — allows browser to send cookies / Authorization headers
        // Required for: session cookie auth, JWT in httpOnly cookie
        // When true, setAllowedOrigins CANNOT be "*" (must be specific origins)

        configuration.setMaxAge(3600L);
        // setMaxAge() — how long (seconds) the browser can cache the preflight result
        // Reduces OPTIONS preflight requests for repeat calls
        // 3600 = 1 hour

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        // UrlBasedCorsConfigurationSource — applies CORS config per URL pattern

        source.registerCorsConfiguration("/**", configuration);
        // registerCorsConfiguration(pattern, config) — apply this config to all URLs
        // You can have different CORS rules for different paths:
        // source.registerCorsConfiguration("/api/**", apiConfig);
        // source.registerCorsConfiguration("/public/**", publicConfig);

        return source;
    }
}
```

#### Per-Controller CORS (Fine-grained)

```java
@RestController
@RequestMapping("/api/products")
@CrossOrigin(origins = "http://localhost:3000")
// @CrossOrigin — enables CORS for this entire controller
// origins — which origins are allowed (can also use * for all)
// Can also be placed on individual methods
public class ProductController {

    @GetMapping
    @CrossOrigin(origins = "*", maxAge = 3600)
    // @CrossOrigin on a method — overrides the controller-level annotation for this endpoint
    // This specific endpoint allows any origin (it's truly public data)
    public List<Product> getProducts() {
        return productService.findAll();
    }

    @PostMapping
    // Uses the controller-level @CrossOrigin config (only localhost:3000)
    public ResponseEntity<Product> createProduct(@RequestBody ProductRequest request) {
        return ResponseEntity.ok(productService.create(request));
    }
}
```

#### Environment-Based CORS Config

In production you don't want localhost as an allowed origin:

```yaml
# application.yml
cors:
  allowed-origins: "https://myapp.com,https://www.myapp.com"

# application-dev.yml (overrides for development)
cors:
  allowed-origins: "http://localhost:3000,https://myapp.com"
```

```java
@Configuration
public class CorsConfig {

    @Value("${cors.allowed-origins}")
    private String[] allowedOrigins;
    // @Value — injects the comma-separated list from yml
    // Spring converts "a,b,c" to String[] automatically

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(Arrays.asList(allowedOrigins));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        // setAllowedHeaders("*") — allow any request header (fine for internal APIs)
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## CSRF + CORS Together in a Real App

A common production setup: React SPA frontend + Spring Boot REST API.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // CORS must be configured FIRST so preflight OPTIONS requests pass
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // REST API with JWT — no sessions, CSRF not needed
            .csrf(csrf -> csrf.disable())

            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            .authorizeHttpRequests(auth -> auth
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()
                // Explicitly permit OPTIONS preflight requests
                // Without this, preflight requests might be blocked before CORS filter
                
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            )

            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        
        // For JWT-based auth, we DON'T need credentials (no cookies)
        // So we can use wildcard origins for public endpoints
        // For protected endpoints, JWTs go in the Authorization header (not affected by CORS credentials)
        config.setAllowedOriginPatterns(List.of("*"));
        // setAllowedOriginPatterns("*") — allows any origin
        // Can use patterns: "https://*.myapp.com" for subdomain wildcard
        
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## Debugging CORS Errors

**Error:** `Access to fetch at 'http://localhost:8080/api' from origin 'http://localhost:3000' has been blocked by CORS policy`

**Checklist:**
1. Is `.cors()` in your `HttpSecurity` config?
2. Is OPTIONS method in `allowedMethods`?
3. Is the frontend's origin in `allowedOrigins`?
4. Is the request header (e.g., Authorization) in `allowedHeaders`?
5. Is your CORS filter before authentication filters? (Spring Security handles this, but double-check)

```java
// Quick debug: log CORS headers on responses
// application.yml:
// logging.level.org.springframework.web.cors: DEBUG
```

---

## Common Mistakes

### CSRF Mistake: Disabling CSRF for cookie-based auth

```java
// Bad — if you use session cookies for auth and disable CSRF,
// CSRF attacks work again
http.csrf(csrf -> csrf.disable()); // Dangerous if you use session cookies!
http.sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED));

// If using sessions + browser clients: KEEP CSRF ENABLED
// If using JWT in headers + no browser form submissions: OK to disable CSRF
```

### CORS Mistake: `allowedOrigins("*")` with `allowCredentials(true)`

```java
// Bad — browsers block this combination; Spring throws an exception
configuration.setAllowedOrigins(List.of("*"));
configuration.setAllowCredentials(true); // Error!
// "When allowCredentials is true, allowedOrigins cannot contain the special value '*'"

// Fix: use explicit origins with credentials
configuration.setAllowedOrigins(List.of("https://myapp.com"));
configuration.setAllowCredentials(true); // OK

// Or use patterns instead of "*"
configuration.setAllowedOriginPatterns(List.of("*")); // Patterns work with credentials
configuration.setAllowCredentials(true); // OK with patterns
```

### CORS Mistake: Configuring CORS in controller but not in Spring Security

```java
// If you have a CORS filter in Spring Security AND @CrossOrigin on controller,
// you might get double CORS headers. Choose one approach.
// Prefer the global CorsConfigurationSource approach for consistency.
```
