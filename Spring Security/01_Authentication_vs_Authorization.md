# Authentication vs Authorization

## The Core Distinction

| Concept | Question it answers | When it happens | Failure result |
|---|---|---|---|
| **Authentication** | "Who are you?" | Before authorization | HTTP 401 Unauthorized |
| **Authorization** | "What can you do?" | After authentication | HTTP 403 Forbidden |

**Real-world analogy:**
- Authentication = Checking your passport at the airport counter. "Are you really John Smith?"
- Authorization = Checking your ticket to decide if you board Business Class or Economy.

You can't authorize someone you haven't authenticated. But just because you authenticated doesn't mean you can access everything.

---

## Authentication in Depth

Authentication is the process of **verifying an identity**. The user claims to be someone; Spring Security verifies that claim.

### The Authentication Flow (Username + Password)

```
Client sends: POST /login  { username: "alice", password: "secret" }
                    |
                    v
        UsernamePasswordAuthenticationFilter
        // Intercepts POST /login
        // Extracts username and password from the request
                    |
                    v
        Creates UsernamePasswordAuthenticationToken(username, password)
        // UsernamePasswordAuthenticationToken — represents an unauthenticated request
        // isAuthenticated() returns false at this point
                    |
                    v
        Passes to AuthenticationManager.authenticate(token)
                    |
                    v
        ProviderManager loops through AuthenticationProviders
                    |
                    v
        DaoAuthenticationProvider.authenticate(token)
        // DaoAuthenticationProvider — the default provider for username/password auth
        // "Dao" because it uses a UserDetailsService (data access object pattern)
                    |
                    v
        Calls userDetailsService.loadUserByUsername("alice")
        // Fetches the user record from your database
                    |
                    v
        Compares stored password hash with provided password
        // Uses PasswordEncoder.matches(rawPassword, encodedPassword)
                    |
              +-----+-----+
              |           |
          PASS          FAIL
              |           |
              v           v
    Returns fully     Throws BadCredentialsException
    populated         --> 401 Unauthorized
    Authentication
    token
              |
              v
    Stored in SecurityContextHolder
    // Now auth.isAuthenticated() == true
```

### Code: What DaoAuthenticationProvider Does Internally

```java
// This is roughly what DaoAuthenticationProvider does behind the scenes.
// You don't write this — Spring writes it. But understanding it helps you debug auth issues.

public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {

    private UserDetailsService userDetailsService;
    // UserDetailsService — your code provides this bean to load users from DB

    private PasswordEncoder passwordEncoder;
    // PasswordEncoder — used to compare raw vs stored (hashed) password

    @Override
    protected UserDetails retrieveUser(String username, 
                                       UsernamePasswordAuthenticationToken authentication) {
        // retrieveUser() — loads the user from your data source
        
        UserDetails loadedUser = userDetailsService.loadUserByUsername(username);
        // loadUserByUsername() — YOUR implementation: query DB, return UserDetails
        
        if (loadedUser == null) {
            throw new InternalAuthenticationServiceException("UserDetailsService returned null");
        }
        return loadedUser;
    }

    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails,
                                                   UsernamePasswordAuthenticationToken authentication) {
        // additionalAuthenticationChecks() — verifies the password
        
        String presentedPassword = authentication.getCredentials().toString();
        // getCredentials() — the raw password the user typed

        if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            // matches() — compares raw password against the stored hash
            throw new BadCredentialsException("Bad credentials");
            // BadCredentialsException — thrown when password doesn't match
        }
    }
}
```

---

## Authorization in Depth

Authorization happens **after** a user is authenticated. Spring Security checks if the authenticated user has the required permissions to access a resource.

### Two Levels of Authorization

#### 1. URL-Level Authorization (most common)

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        
        http.authorizeHttpRequests(auth -> auth
            
            // --- Public endpoints (no login required) ---
            .requestMatchers("/api/public/**").permitAll()
            // permitAll() — anyone, authenticated or not, can access
            
            .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
            // Also allow anonymous GET on products (read-only public catalog)
            
            // --- Role-based access ---
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            // hasRole("ADMIN") — user must have ROLE_ADMIN authority
            // Spring automatically prepends "ROLE_" — so you write "ADMIN" not "ROLE_ADMIN"
            
            .requestMatchers("/api/manager/**").hasAnyRole("ADMIN", "MANAGER")
            // hasAnyRole() — user must have at least one of the listed roles
            
            // --- Authority-based access (more fine-grained than roles) ---
            .requestMatchers(HttpMethod.DELETE, "/api/products/**").hasAuthority("PRODUCT_DELETE")
            // hasAuthority() — requires this exact authority string (no "ROLE_" prefix added)
            
            .requestMatchers(HttpMethod.POST, "/api/products/**").hasAuthority("PRODUCT_CREATE")
            
            // --- Catch-all ---
            .anyRequest().authenticated()
            // authenticated() — user must be logged in, any role is fine
        );
        
        return http.build();
    }
}
```

#### 2. Method-Level Authorization

Placed directly on service methods using annotations. Covered in depth in `07_Method_Security.md`.

```java
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    // @PreAuthorize — checks the condition BEFORE the method runs
    // If condition is false, throws AccessDeniedException (403)
    public void deleteUser(Long userId) {
        userRepository.deleteById(userId);
    }

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    // SpEL expression — admin can access anyone's profile, user can only access their own
    // #userId — refers to the method parameter named userId
    // authentication.principal.id — the logged-in user's ID from SecurityContext
    public UserProfile getProfile(Long userId) {
        return userRepository.findById(userId).orElseThrow();
    }
    
    @PostAuthorize("returnObject.ownerId == authentication.principal.id")
    // @PostAuthorize — checks AFTER the method runs, using the return value
    // returnObject — the object returned by the method
    // Use when you need to check the actual data returned, not just input params
    public Document getDocument(Long docId) {
        return documentRepository.findById(docId).orElseThrow();
    }
}
```

---

## Roles vs Authorities — The Confusion Explained

This trips up almost every developer new to Spring Security.

```
ROLE_ADMIN          <-- this is a GrantedAuthority (string)
ROLE_USER           <-- this is a GrantedAuthority (string)
PRODUCT_DELETE      <-- this is a GrantedAuthority (string)
```

**Roles** are just authorities that follow the `ROLE_` prefix convention.

```java
// hasRole("ADMIN")       checks for authority "ROLE_ADMIN"
// hasAuthority("ADMIN")  checks for authority "ADMIN" exactly (no prefix added)

// So these two are equivalent:
.hasRole("ADMIN")                    // Spring adds "ROLE_" prefix internally
.hasAuthority("ROLE_ADMIN")          // You provide the full string

// And these two are NOT equivalent:
.hasRole("PRODUCT_DELETE")           // checks for "ROLE_PRODUCT_DELETE" — wrong!
.hasAuthority("PRODUCT_DELETE")      // checks for "PRODUCT_DELETE" exactly — correct
```

**Best practice in production:** Use `ROLE_` prefix for coarse-grained roles (ADMIN, USER, MANAGER) and plain authorities for fine-grained permissions (PRODUCT_READ, ORDER_CANCEL).

---

## The AccessDecisionManager (How Authorization is Decided)

When a request hits a secured resource, Spring Security uses an `AccessDecisionManager`:

```
Request arrives at secured endpoint
            |
            v
    FilterSecurityInterceptor
    // Intercepts the request at the last filter in the chain
            |
            v
    Collects ConfigAttributes (the required permissions for this URL)
    // e.g., ["ROLE_ADMIN"] for /admin/**
            |
            v
    AccessDecisionManager.decide(authentication, resource, configAttributes)
    // decide() — checks if authentication has the required configAttributes
            |
       +----+----+
       |         |
    GRANTED    DENIED
       |         |
       v         v
  Request    AccessDeniedException
  continues  --> 403 Forbidden
```

---

## Practical Example: E-Commerce REST API

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
// @EnableMethodSecurity — enables @PreAuthorize, @PostAuthorize, @Secured on methods
// Must be added to use method-level security annotations
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            // csrf.disable() — disabling CSRF for REST APIs (explained in 08_CSRF_and_CORS.md)
            // REST APIs are stateless; CSRF attacks target session-based apps
            
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // SessionCreationPolicy.STATELESS — no HTTP session, each request is independent
            // Required for JWT-based APIs
            
            .authorizeHttpRequests(auth -> auth
                // Public: browsing the catalog doesn't require login
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                
                // Customers: place orders, view own profile
                .requestMatchers(HttpMethod.POST, "/api/orders/**").hasRole("CUSTOMER")
                .requestMatchers("/api/profile/**").hasRole("CUSTOMER")
                
                // Admins: manage products, view all orders
                .requestMatchers(HttpMethod.POST, "/api/products/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/products/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.GET, "/api/orders/**").hasRole("ADMIN")
                
                // Everything else requires login
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
}

// The controller doesn't need to check roles — security handles it declaratively
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    // This is only reachable if the user has ROLE_CUSTOMER — enforced by the filter chain
    public ResponseEntity<Order> placeOrder(@RequestBody OrderRequest request) {
        // No manual role check needed here
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String customerId = auth.getName();
        // getName() — returns the username/principal identifier of the current user
        
        Order order = orderService.placeOrder(customerId, request);
        return ResponseEntity.ok(order);
    }
}
```

---

## Common Mistakes

### Mistake 1: Confusing 401 and 403

```
401 Unauthorized  → User is NOT authenticated (no login / bad credentials)
403 Forbidden     → User IS authenticated but doesn't have permission
```

If your API returns 401 when a logged-in user hits a restricted endpoint, your auth filter isn't storing the authentication correctly.

### Mistake 2: Using `hasRole` for fine-grained permissions

```java
// Bad — too coarse
.requestMatchers(HttpMethod.DELETE, "/api/products/**").hasRole("ADMIN")
// Now ALL admins can delete products, even finance admins who should not

// Better — fine-grained authority
.requestMatchers(HttpMethod.DELETE, "/api/products/**").hasAuthority("PRODUCT_DELETE")
// Only users explicitly granted PRODUCT_DELETE can delete
```

### Mistake 3: Putting business logic in `UserDetailsService`

```java
// Bad — UserDetailsService should ONLY load user data
public UserDetails loadUserByUsername(String username) {
    User user = userRepository.findByUsername(username);
    user.setLastLoginTime(LocalDateTime.now()); // Side effect — don't do this
    return buildUserDetails(user);
}

// Good — just load and map, nothing else
public UserDetails loadUserByUsername(String username) {
    User user = userRepository.findByUsername(username)
        .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        // UsernameNotFoundException — throw this, not null, when user doesn't exist
    return buildUserDetails(user);
}
```
